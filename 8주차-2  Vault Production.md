# 8주차-2 Vault Production

## 목차

* [1. Vault 개요 (Secrets / Policy / Auth)](#1-vault-개요-secrets--policy--auth)
* [2. Vault HA 구성 (kind + Helm + Raft)](#2-vault-ha-구성-kind--helm--raft)
* [3. KV Secret 엔진 실습 (쓰기/읽기/목록)](#3-kv-secret-엔진-실습-쓰기읽기목록)
* [4. Policy를 이용한 Secret 접근 제한](#4-policy를-이용한-secret-접근-제한)
* [5. Vault REST API + curl 실습](#5-vault-rest-api--curl-실습)
* [6. OpenLDAP 구축 및 디렉토리 구조](#6-openldap-구축-및-디렉토리-구조)
* [7. Vault LDAP Auth 구성 및 토큰/정책 확인](#7-vault-ldap-auth-구성-및-토큰정책-확인)
* [8. LDAP 그룹을 Vault Policy에 매핑](#8-ldap-그룹을-vault-policy에-매핑)
* [9. Vault TLS 구성 (CA/서버 인증서 + Helm)](#9-vault-tls-구성-ca서버-인증서--helm)
* [10. ingress-nginx SSL Passthrough로 Vault 노출](#10-ingress-nginx-ssl-passthrough로-vault-노출)

---

## 1. Vault 개요 (Secrets / Policy / Auth)

### 1.1 Secrets

* Vault는 여러 유형의 비밀을 **Secret Engine** 단위로 관리
* Secret Engine의 역할

  * 데이터 저장: 암호화된 KV 저장소, Redis/Memcached 유사 저장소 등
  * 데이터 생성: 동적 자격증명(DB 계정, 클라우드 IAM 등)
  * 서비스로서의 암호화: 암호화/복호화 API 제공
  * 기타: TOTP, PKI, 인증서 발급 등
* Secret Engine은 **특정 경로(path)에 마운트**되며, 파일 시스템 디렉터리처럼 계층 구조를 가짐

### 1.2 Policy

* Vault의 모든 요청은 **정책(Policy)** 에 의해 허용 또는 거부
* 기본 철학: **기본 거부(deny by default)**

  * 빈 정책은 어떤 권한도 부여하지 않는다.
* 정책은 **경로 기반(path-based)** 으로 작성한다.

  * `secret/data/*` 처럼 와일드카드를 사용한 범위 정책 가능
  * 매우 세분화된 특정 경로에만 허용/거부도 가능

### 1.3 Authentication과 Token

* Vault의 모든 동작은 **토큰(Token)** 을 통해 보호
* 토큰은 하나 이상의 **정책**을 갖고 있으며, 정책을 통해

  * 어떤 Secret Engine에 접근 가능한지
  * 어떤 관리 기능을 수행 가능한지 결정
* 토큰 수명

  * TTL, 최대 TTL, 갱신 가능 여부 등이 설정되어 **유출/오남용 위험을 줄이는 구조**
* 인증(예: LDAP, Kubernetes, GitHub 등)을 성공하면

  * Vault가 토큰을 발급하고, 정책을 부여한 뒤 클라이언트에 반환

---

## 2. Vault HA 구성 (kind + Helm + Raft)

### 2.1 kind 클러스터 및 ingress-nginx 구성

```bash
kind create cluster --name myk8s --image kindest/node:v1.32.8 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  labels:
    ingress-ready: true
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
- role: worker
- role: worker
- role: worker
EOF

# ingress-nginx 배포
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# ingress 전용 노드 선택
kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
  --type='merge' \
  -p='{
    "spec": {
      "template": {
        "spec": {
          "nodeSelector": {
            "ingress-ready": "true"
          }
        }
      }
    }
  }'

# SSL Passthrough 활성화
kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml \
| sed '/- --publish-status-address=localhost/a\\
        - --enable-ssl-passthrough' | kubectl apply -f -
```

### 2.2 Vault HA Helm values (Raft 기반)

```bash
kubectl create ns vault

cat << EOF > values-ha.yaml
server:
  replicas: 3

  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
    config: |
      ui = true
      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }

      service_registration "kubernetes" {}

  readinessProbe:
    enabled: true

  dataStorage:
    enabled: true
    size: 10Gi

  service:
    enabled: true
    type: ClusterIP
    port: 8200
    targetPort: 8200

ui:
  enabled: true
  serviceType: "NodePort"
  externalPort: 8200
  serviceNodePort: 30000

injector:
  enabled: false
EOF

helm install vault hashicorp/vault -n vault -f values-ha.yaml --version 0.31.0
```

* `server.ha.raft.enabled = true` 로 **내장 Raft 스토리지 + HA** 구성
* `dataStorage` 로 각 Pod에 PV를 붙여 Raft 데이터 영속화

### 2.3 초기화(init), Unseal, Raft Join 플로우

1. `vault-0`에서 `vault operator init` 실행

   * Shamir Secret Sharing으로 Unseal Key 5개, Threshold 3 생성
   * Root Token 발급
2. `vault operator unseal` 3회 실행하여 `vault-0`을 활성화
3. `vault-1`, `vault-2`에서 `vault operator raft join http://vault-0.vault-internal:8200`

   * Raft 클러스터에 합류
4. 각 노드에서 다시 `vault operator unseal` 3회
5. `vault operator raft list-peers` 로 leader/follower 상태 확인

---

## 3. KV Secret 엔진 실습 (쓰기/읽기/목록)

### 3.1 KV v2 Secret Engine 활성화

```bash
# mysecret 경로에 kv-v2 엔진 마운트
vault secrets enable -path=mysecret kv-v2
```

### 3.2 Secret 작성 및 확인

```bash
# 시크릿 저장
vault kv put mysecret/logins/study \
  username="demo" \
  password="p@ssw0rd"

# 조회
vault kv get mysecret/logins/study

# 목록 확인
vault kv list mysecret
vault kv list mysecret/logins
```

* 경로 설계 예: `mysecret/logins/<provider>`
* Vault는 URI 규칙만 만족하면 경로 구조에 제약을 두지 않음

---

## 4. Policy를 이용한 Secret 접근 제한

### 4.1 특정 Secret에 대한 deny 정책 예시

```bash
vault policy write restrict_study - <<EOF
path "mysecret/logins/study*" {
  capabilities = ["deny"]
}
EOF
```

* 해당 정책이 붙은 토큰은 `mysecret/logins/study` 및 하위 경로에 **접근 불가**
* 다른 경로에 대해서는 별도 정책으로 허용 여부를 제어

---

## 5. Vault REST API + curl 실습

### 5.1 CLI와 API 경로 차이

* CLI: `vault kv get mysecret/logins/study`
* API: `GET /v1/mysecret/data/logins/study`

  * KV v2인 경우 `.../data/...` 하위 경로를 사용해야 함

### 5.2 curl + jq 예제

```bash
export VAULT_ROOT_TOKEN=hvs.uHKVZKlswZ0wKsp1tMA3DZOs

curl -s \
  --header "X-Vault-Token: $VAULT_ROOT_TOKEN" \
  http://localhost:30000/v1/mysecret/data/logins/study | jq

# username, password만 추출
curl -s --header "X-Vault-Token: $VAULT_ROOT_TOKEN" \
  http://localhost:30000/v1/mysecret/data/logins/study \
  | jq -r .data.data.username

curl -s --header "X-Vault-Token: $VAULT_ROOT_TOKEN" \
  http://localhost:30000/v1/mysecret/data/logins/study \
  | jq -r .data.data.password

# 환경변수에 저장
export USER_NAME=$(curl -s --header "X-Vault-Token: $VAULT_ROOT_TOKEN" \
  http://localhost:30000/v1/mysecret/data/logins/study | jq -r .data.data.username)

export PASSWORD=$(curl -s --header "X-Vault-Token: $VAULT_ROOT_TOKEN" \
  http://localhost:30000/v1/mysecret/data/logins/study | jq -r .data.data.password)
```

* CLI는 `~/.vault-token` 파일을 자동 사용하지만, API는 반드시 `X-Vault-Token` 헤더에 토큰을 포함해야 함

---

## 6. OpenLDAP 구축 및 디렉토리 구조

### 6.1 OpenLDAP + phpLDAPadmin 배포

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openldap
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openldap
  namespace: openldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openldap
  template:
    metadata:
      labels:
        app: openldap
    spec:
      containers:
        - name: openldap
          image: osixia/openldap:1.5.0
          ports:
            - containerPort: 389
              name: ldap
            - containerPort: 636
              name: ldaps
          env:
            - name: LDAP_ORGANISATION
              value: "Example Org"
            - name: LDAP_DOMAIN
              value: "example.org"
            - name: LDAP_ADMIN_PASSWORD
              value: "admin"
            - name: LDAP_CONFIG_PASSWORD
              value: "admin"
        - name: phpldapadmin
          image: osixia/phpldapadmin:0.9.0
          ports:
            - containerPort: 80
              name: phpldapadmin
          env:
            - name: PHPLDAPADMIN_HTTPS
              value: "false"
            - name: PHPLDAPADMIN_LDAP_HOSTS
              value: "openldap"
---
apiVersion: v1
kind: Service
metadata:
  name: openldap
  namespace: openldap
spec:
  selector:
    app: openldap
  ports:
    - name: phpldapadmin
      port: 80
      targetPort: 80
      nodePort: 30001
    - name: ldap
      port: 389
      targetPort: 389
    - name: ldaps
      port: 636
      targetPort: 636
  type: NodePort
EOF
```

* 기본 정보

  * Base DN: `dc=example,dc=org`
  * Bind DN: `cn=admin,dc=example,dc=org`
  * Password: `admin`

### 6.2 최종 트리 구조

```text
dc=example,dc=org
├── ou=people
│   ├── uid=alice
│   └── uid=bob
└── ou=groups
    ├── cn=devs
    └── cn=admins
```

* users: `alice/alice123`, `bob/bob123`
* groups:

  * `devs` (member: bob)
  * `admins` (member: alice)

### 6.3 ldapadd 명령으로 객체 생성

* `ou=people`, `ou=groups` 생성
* `uid=alice`, `uid=bob` (inetOrgPerson)
* `cn=devs`, `cn=admins` (groupOfNames)

해당 부분은 실습에서 사용한 `ldapadd` 스크립트 그대로 캔버스에 보존.

---

## 7. Vault LDAP Auth 구성 및 토큰/정책 확인

### 7.1 LDAP Auth Method 활성화 및 설정

```bash
# LDAP auth enable
vault auth enable ldap

# LDAP 설정
vault write auth/ldap/config \
  url="ldap://openldap.openldap.svc:389" \
  starttls=false \
  insecure_tls=true \
  binddn="cn=admin,dc=example,dc=org" \
  bindpass="admin" \
  userdn="ou=people,dc=example,dc=org" \
  groupdn="ou=groups,dc=example,dc=org" \
  groupfilter="(member=uid={{.Username}},ou=people,dc=example,dc=org)" \
  groupattr="cn"
```

### 7.2 LDAP 사용자로 로그인 및 토큰 확인

```bash
vault login -method=ldap username=alice
# Password: alice123
```

* `vault token lookup` 결과 주요 필드

  * `display_name = ldap-alice`
  * `meta.username = alice`
  * `policies = [default]`
  * `token_policies`: 토큰에 직접 부여된 정책 목록
  * `identity_policies`: Identity(Entity/Group)에서 계산된 정책
  * `policies`: 위 둘의 합집합 (실제 권한 평가 기준)

`bob`도 동일하게 `vault login -method=ldap username=bob password=bob123` 로 테스트.

---

## 8. LDAP 그룹을 Vault Policy에 매핑

### 8.1 admin 정책 정의

```bash
vault policy write admin - <<EOF
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF

vault policy read admin
```

### 8.2 LDAP 그룹과 정책 연결

```bash
vault write auth/ldap/groups/admins policies=admin
```

* `admins` 그룹에 속한 `alice`가 LDAP로 로그인하면

  * `token_policies = ["admin" "default"]`
  * `policies = ["admin" "default"]`
* `vault policy read admin` 으로 실제 권한 확인 가능.

---

## 9. Vault TLS 구성 (CA/서버 인증서 + Helm)

### 9.1 kind 클러스터 및 ingress-nginx 재구성

```bash
kind create cluster --name myk8s --image kindest/node:v1.32.8 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  labels:
    ingress-ready: true
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
EOF

kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml \
| sed '/- --publish-status-address=localhost/a\\
        - --enable-ssl-passthrough' | kubectl apply -f -

kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
  --type='merge' \
  -p='{
    "spec": {
      "template": {
        "spec": {
          "nodeSelector": {
            "ingress-ready": "true"
          }
        }
      }
    }
  }'
```

### 9.2 CA / 서버 인증서 생성

```bash
docker exec -it myk8s-control-plane bash

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

mkdir /tls && cd /tls

# CA
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes \
  -key ca.key \
  -subj "/CN=Vault-CA" \
  -days 3650 \
  -out ca.crt

# 서버 키/CSR
openssl genrsa -out vault.key 2048
openssl req -new -key vault.key \
  -subj "/CN=vault.example.com" \
  -out vault.csr

# 서버 인증서 발급 (SAN 포함)
openssl x509 -req \
  -in vault.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out vault.crt \
  -days 3650 \
  -extensions v3_req \
  -extfile <(cat <<EOF
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = vault.example.com
DNS.2 = vault
DNS.3 = localhost
DNS.4 = vault.vault.svc.cluster.local
DNS.5 = vault.vault-internal
DNS.6 = vault-0.vault-internal
DNS.7 = vault-1.vault-internal
DNS.8 = vault-2.vault-internal
DNS.9 = 127.0.0.1
EOF
)
```

### 9.3 TLS Secret 및 Helm values-tls.yaml

```bash
kubectl create namespace vault

kubectl -n vault create secret tls vault-tls \
  --cert=vault.crt \
  --key=vault.key

kubectl -n vault create secret generic vault-ca \
  --from-file=ca.crt=ca.crt

cat << EOF > values-tls.yaml
global:
  tlsDisable: false

injector:
  enabled: false

server:
  volumes:
    - name: vault-tls
      secret:
        secretName: vault-tls
    - name: vault-ca
      secret:
        secretName: vault-ca
  volumeMounts:
    - name: vault-tls
      mountPath: /vault/user-tls
      readOnly: true
    - name: vault-ca
      mountPath: /vault/user-ca
      readOnly: true

  standalone:
    enabled: "true"
    config: |-
      ui = true
      listener "tcp" {
        tls_disable = 0
        address = "0.0.0.0:8200"
        cluster_address = "0.0.0.0:8201"
        tls_cert_file = "/vault/user-tls/tls.crt"
        tls_key_file  = "/vault/user-tls/tls.key"
      }
      storage "file" {
        path = "/vault/data"
      }
      api_addr    = "https://vault.vault.svc.cluster.local:8200"
      cluster_addr = "https://vault-0.vault-internal:8201"
EOF

helm install vault hashicorp/vault -n vault -f values-tls.yaml --version 0.29.0
```

### 9.4 init / unseal 및 NodePort

```bash
kubectl exec vault-0 -n vault -- \
  vault operator init -tls-skip-verify \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json

VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)

kubectl exec vault-0 -n vault -- \
  vault operator unseal -tls-skip-verify $VAULT_UNSEAL_KEY

# 서비스 NodePort로 노출
kubectl patch svc -n vault vault \
  -p '{"spec":{"type":"NodePort","ports":[{"port":8200,"targetPort":8200,"nodePort":30000}]}}'

export VAULT_ADDR='https://localhost:30000'

vault status -tls-skip-verify
vault login -tls-skip-verify
```

---

## 10. ingress-nginx SSL Passthrough로 Vault 노출

### 10.1 hosts 설정 및 포트 확인

```bash
echo "127.0.0.1 vault.example.com" | sudo tee -a /etc/hosts

kubectl describe pod -n vault vault-0 | grep -i ports
# 8200/TCP (https), 8201/TCP (https-internal), 8202/TCP (https-rep)
```

### 10.2 Ingress 리소스 생성

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-https
  namespace: vault
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: "nginx"
  rules:
    - host: vault.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vault
                port:
                  number: 8200
  tls:
    - hosts:
        - vault.example.com
      secretName: vault-tls
EOF

kubectl get ingress -n vault
```

* 브라우저에서 `https://vault.example.com` 접속 후 UI로 관리 가능.
