# 8주차 - HashiCorp Vault / VSO on Kubernetes

## 목차

* [1. Vault 설치 on K8S (kind)](#1-vault-설치-on-k8s-kind)
* [2. KIND 클러스터 생성](#2-kind-클러스터-생성)
* [3. Helm 기반 Vault 설치](#3-helm-기반-vault-설치)
* [4. Vault 초기화init--unseal](#4-vault-초기화init--unseal)
* [5. Vault CLI 로그인 (로컬)](#5-vault-cli-로그인-로컬)
* [6. Vault UI 접속](#6-vault-ui-접속)
* [7. Vault Audit Log 설정 (선택)](#7-vault-audit-log-설정-선택)
* [8. Vault 사용 on K8S](#8-vault-사용-on-k8s)

---

## 1. Vault 설치 on K8S (kind)

* Kubernetes 환경(kind)에 Vault를 **설치**
* Vault 서버를 **Initialize → Unseal → Login**까지 수행
* NodePort 기반으로 **Vault UI 및 CLI 접근**을 구성

---

## 2. KIND 클러스터 생성

Vault 실습을 위한 가장 간단한 형태의 K8S 환경을 kind 로 구성

### 중요 Key Point

* Vault UI(NodePort: 30000) 사용을 위해 포트 매핑 필요
* Control-plane 노드에 실습용 유틸리티 설치 (ping, tcpdump, vim, git, lsof 등)

### 설치

```bash
kind create cluster --name myk8s --image kindest/node:v1.32.8 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  labels:
    ingress-ready: true
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
EOF

kubectl get node

docker exec -it myk8s-control-plane sh -c 'apt update && apt install tree psmisc lsof wget net-tools dnsutils tcpdump ngrep iputils-ping git vim -y'
```

---

## 3. Helm 기반 Vault 설치

HashiCorp 공식 Helm chart 를 사용

### 중요 중요 개념

* `standalone` 모드는 개발/테스트용으로 간단히 사용 가능
* `tlsDisable: true` → 학습 목적의 비 TLS 구성
* Storage: file (데이터) / logs (audit)
* NodePort 로 UI 접근 가능

### `vault-values.yaml`

```yaml
global:
  enabled: true
  tlsDisable: true

server:
  standalone:
    enabled: true
    config: |
      ui = true
      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_disable = 1
      }
      storage "file" {
        path = "/vault/data"
      }

  dataStorage:
    enabled: true
    size: "10Gi"
    mountPath: "/vault/data"

  auditStorage:
    enabled: true
    size: "10Gi"
    mountPath: "/vault/logs"

  service:
    enabled: true
    type: NodePort
    nodePort: 30000

ui:
  enabled: true

injector:
  enabled: false
```

### 설치

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
kubectl create namespace vault

helm upgrade vault hashicorp/vault -n vault -f vault-values.yaml --install --version 0.31.0
```

### 상태 확인

```bash
kubectl get sts,pods,svc,pvc -n vault
kubectl exec -ti vault-0 -n vault -- vault status
```

---

## 4. Vault 초기화(Init) & Unseal

Vault 는 **보안상 기본적으로 Sealed** 상태로 시작
Unseal 을 진행해야 정상 동작하며, 이 과정을 이해하는 것이 중요

### 개념

* **Init**: Root Token 과 Unseal Key 생성
* **Shamir Secret Sharing** 방식 사용
* Key-Share = 1, Key-Threshold = 1 구성(학습용)
* Unseal 과정 후 Vault 서버는 **Ready** 상태가 됨

### 설치

```bash
kubectl exec vault-0 -n vault -- vault operator init \
  -key-shares=1 \
  -key-threshold=1 \
  -format=json > cluster-keys.json

cat cluster-keys.json | jq
```

### 설치 Unseal

```bash
VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)
kubectl exec vault-0 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
```

### 상태 확인

```bash
kubectl get pod -n vault
```

### Root Token 추출

```bash
jq -r ".root_token" cluster-keys.json
```

---

## 5. Vault CLI 로그인 (로컬)

```bash
export VAULT_ADDR='http://localhost:30000'
vault status
vault login   # root token 입력
```

---

## 6. Vault UI 접속

```bash
kubectl get svc vault -n vault
open http://127.0.0.1:30000
```

Token 인증 방식으로 로그인한다.

---

## 7. Vault Audit Log 설정 (선택)

Audit 로그는 Vault 운영에서 반드시 활성화해야 하는 필수 기능.
PVC 기반 file audit log 사용.

### 중요

* **Audit 로그 PVC 용량이 가득 차면 Vault 동작 자체가 중단됨!**
* Logger 수준이 아니라 Vault 엔진 동작과 직접 연결

### 설정

```bash
vault audit enable file file_path=/vault/logs/audit.log
vault audit list -detailed
kubectl exec -it vault-0 -n vault -- tail -f /vault/logs/audit.log
```

---

## 8. Vault 사용 on K8S

### 목적 목적

* Vault에 **시크릿을 생성**하고,
* Kubernetes 파드(웹 애플리케이션)가 **서비스 어카운트 토큰(JWT)** 으로 Vault에 **로그인 → 시크릿을 조회**하는 전체 흐름 실습

### 8.1 전체 흐름 개요 (Kubernetes Auth)

참고: [https://developer.hashicorp.com/vault/tutorials/kubernetes/agent-kubernetes](https://developer.hashicorp.com/vault/tutorials/kubernetes/agent-kubernetes)

1. **(1) Vault에 Role/Policy 설정**

   * 애플리케이션이 어떤 경로의 시크릿을 읽을 수 있는지 **Policy** 정의
   * 해당 Policy를 사용하는 **Role** 을 Kubernetes Auth 메서드에 설정

2. **(2) 파드 생성 시 ServiceAccount 토큰(JWT) 생성**

   * 파드에 바인딩된 ServiceAccount 기준으로 **SAT(ServiceAccountToken)** 발급

3. **(3) 파드 애플리케이션의 Vault 로그인 과정**
   3-1) 앱이 **JWT를 Vault로 전송**하여 `/auth/kubernetes/login` 호출
   3-2) Vault는 **TokenReview API** 로 Kubernetes API 서버에 토큰 검증 요청
   3-3) K8S API 서버는 **서비스 어카운트 이름, 네임스페이스** 를 반환
   3-4) Vault는 이 정보를 Role/Policy 와 매칭하여 허용 여부 판단
   3-5) 성공 시 Vault는 앱에 **Vault Auth Token** 을 발급

4. **(4) 애플리케이션이 Vault 시크릿을 요청**
   4-1) 앱은 발급받은 **Vault Token** 으로 `secret/data/...` 경로에 시크릿 조회 요청
   4-2) Vault는 토큰의 Policy 확인 후 허용 여부 판별
   4-3) 허용 시 최종 **시크릿 데이터(username, password 등)** 반환

> 이 **Kubernetes Auth 흐름(k8s-auth)** 는 뒤에서 다룰 **VSO(Vault Secrets Operator)** 에서도 그대로 활용
> AWS EKS의 `aws-auth` 매커니즘에서도 유사한 토큰 검증/매핑 개념을 사용

---

### 8.2 Vault에 시크릿 생성 (kv-v2)

```bash
# kv-v2 시크릿 엔진 활성화
vault secrets enable -path=secret kv-v2

# 확인
vault secrets list

# 시크릿 생성 (username / password)
vault kv put secret/webapp/config \
  username="static-user" \
  password="static-password"

# 시크릿 조회
vault kv get secret/webapp/config

# Root Token 환경변수 설정 후 HTTP API 로도 확인
export VAULT_ROOT_TOKEN="<root-token>"

curl -s --header "X-Vault-Token: $VAULT_ROOT_TOKEN" \
  --request GET \
  http://127.0.0.1:30000/v1/secret/data/webapp/config | jq
```

(옵션) Web UI 에서도 `secret/webapp/config` 경로로 동일 시크릿을 확인 가능.

---

### 8.3 Kubernetes Authentication 구성 (Vault 쪽 설정)

**구성 관계도**
`[Auth] k8s role(webapp)` ⟵ `[Policy] path secret/data/webapp/config read` ⟵ `[Secret] username/password`

#### 8.3.1 Vault가 K8S 토큰을 검증할 수 있는지 확인

```bash
# vault 서버 서비스어카운트가 가진 RBAC 확인
kubectl rbac-tool lookup vault
kubectl rolesum vault -n vault
```

* `system:auth-delegator` ClusterRole 을 통해

  * `subjectaccessreviews.authorization.k8s.io`
  * `tokenreviews.authentication.k8s.io`
    를 수행할 수 있어야 함 (TokenReview 기반 검증).

#### 8.3.2 Kubernetes Auth 메서드 활성화 및 설정

```bash
# Kubernetes auth 활성화
vault auth enable kubernetes

# 설정 확인
vault auth list

# K8S API 서버 엔드포인트 설정
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"

# 설정 확인
vault read auth/kubernetes/config
```

#### 8.3.3 webapp용 Policy 생성

```bash
# secret/data/webapp/config 에 read 권한 부여
vault policy write webapp - <<EOF
path "secret/data/webapp/config" {
  capabilities = ["read"]
}
EOF
```

#### 8.3.4 Kubernetes Auth Role(webapp) 생성

```bash
vault write auth/kubernetes/role/webapp \
  bound_service_account_names=vault \
  bound_service_account_namespaces=default \
  policies=webapp \
  ttl=24h \
  audience="https://kubernetes.default.svc.cluster.local"
```

* SA `default/vault` 로부터 오는 토큰만 **role=webapp** 으로 매핑
* 해당 토큰에 `webapp` Policy 부여, TTL 24시간

---

### 8.4 사전 지식 정리: ServiceAccount, SAT, JWT, OIDC

#### 8.4.1 ServiceAccount & ServiceAccountToken

* **ServiceAccount(SA)**

  * 파드에서 실행되는 프로세스의 **ID** 역할
  * 파드는 SA에 바인딩된 토큰으로 K8S API 서버에 인증

* **ServiceAccountToken(SAT)**

  * kubelet 이 TokenRequest API 로 발급받는 토큰
  * 파드 라이프사이클 또는 만료 시간(기본 1시간)까지 유효
  * 특정 파드 & API 서버를 대상으로 바인딩됨

* **토큰 컨트롤러 / ServiceAccount Admission Controller** 는

  * SA 생성/삭제에 따라 토큰 시크릿/볼륨을 관리
  * 파드 생성 시 `serviceAccountName` 기본값을 `default` 로 설정
  * 필요한 경우 `/var/run/secrets/kubernetes.io/serviceaccount` 에 토큰을 마운트

#### 8.4.2 ServiceAccount Token Volume Projection (PSAT)

기존 **시크릿 볼륨** 대신 **Projected Volume** 으로 토큰/CA/namespace 정보를 한 디렉터리에 투사.

예시:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  serviceAccountName: build-robot
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```

K8S 1.22+ 에서는 기본적으로 **bound service account token volume** 형태로
`kube-api-access-<suffix>` projected volume 이 자동 추가됨

#### 8.4.3 JWT & OIDC 짧은 정리

* **JWT(JSON Web Token)**

  * Header + Payload + Signature 구조
  * private key 로 서명, public key로 검증
  * Bearer 토큰을 **위조 방지** 가능한 구조로 만든 JSON 기반 포맷

* **OIDC(OpenID Connect)**

  * OpenID(인증) + OAuth2.0(인가)을 합친 프로토콜
  * IdP(예: Google, Keycloak)가 ID Token / Access Token 발급
  * K8S, Vault, Keycloak 등을 연동할 때 핵심 개념.

---

### 8.5 WebApp 배포: Vault와 연동하는 예제 애플리케이션

예제 웹앱 동작:

* HTTP 요청 수신 → 파드 안의 **ServiceAccount JWT** 읽기
* `/auth/kubernetes/login` 으로 Vault 로그인 → Vault 토큰 획득
* `secret/data/webapp/config` 시크릿 조회 후 HTTP 응답으로 username/password 출력

#### 8.5.1 webapp 배포 매니페스트

```bash
kubectl create sa vault

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      serviceAccountName: vault
      containers:
        - name: app
          image: hashieducation/simple-vault-client:latest
          imagePullPolicy: Always
          env:
            - name: VAULT_ADDR
              value: 'http://vault.vault.svc:8200'
            - name: JWT_PATH
              value: '/var/run/secrets/kubernetes.io/serviceaccount/token'
            - name: SERVICE_PORT
              value: '8080'
          volumeMounts:
          - name: sa-token
            mountPath: /var/run/secrets/kubernetes.io/serviceaccount
            readOnly: true
      volumes:
      - name: sa-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 600
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    nodePort: 30001
EOF

kubectl get pod -l app=webapp
```

#### 8.5.2 SA 토큰 및 JWT Payload 확인

```bash
# 토큰 내용 확인 (10분마다 갱신)
kubectl exec -it deploy/webapp -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Payload 디코딩
kubectl exec -it deploy/webapp -- \
  sh -c 'cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d "." -f2 | base64 -d; echo ""'
```

#### 8.5.3 애플리케이션 동작 확인

```bash
curl 127.0.0.1:30001
# 출력 예시
# password:static-password username:static-user
```

Vault의 시크릿을 수정하면 응답이 바뀌는 것도 확인 가능:

```bash
vault kv put secret/webapp/config \
  username="changed-user" \
  password="changed-password"

vault kv get secret/webapp/config

curl 127.0.0.1:30001
# password:changed-password username:changed-user
```

---

### 8.6 트래픽 & Audit Log 로 흐름 검증

#### 8.6.1 webapp → Vault 네트워크 트래픽 (ngrep)

```bash
# veth0600eea7: webapp 파드에 연결된 veth 라고 가정
ngrep -tW byline -d veth0600eea7 '' 'tcp port 8200'
```

관찰 포인트:

1. `POST /v1/auth/kubernetes/login`

   * Body 에 `{"role":"webapp","jwt":"<서비스어카운트 JWT>"}`
   * 응답 200 OK, Body 에 Vault 토큰 포함
2. `GET /v1/secret/data/webapp/config`

   * Header `X-Vault-Token: <발급 토큰>`
   * 응답 200 OK, Body 에 username/password 포함

#### 8.6.2 Vault Audit 로그에서 흐름 확인

```bash
kubectl exec -it vault-0 -n vault -- \
  tail -f /vault/logs/audit.log
```

관찰 포인트:

* `path="auth/kubernetes/login"` + `type="request" | "response"`

  * JWT, role 값은 HMAC 으로 마스킹
  * response.auth.client_token 에 발급된 토큰 해시 표시
* `path="secret/data/webapp/config"` + `operation="read"`

  * `webapp` Policy 에 의해 허용됨을 `policy_results` 로 확인

Audit 로그는 **보안/운영 측면에서 강력한 추적 근거**를 제공하므로,
실 서비스 환경에서는 **반드시 2개 이상의 audit 디바이스**(file + syslog 등)를 운영해야 함

---

### 8.7 정리: K8S 앱이 Vault 시크릿을 사용하는 패턴

1. 앱이 사용할 시크릿을 **Vault kv-v2** 에 저장
2. 해당 시크릿 경로에 대해 최소 권한 **Policy** 작성
3. Policy 를 참조하는 **Kubernetes Auth Role** 생성
4. Role 에 바인딩될 **ServiceAccount** 및 Namespace 지정
5. 파드에 해당 SA 를 설정하고, **JWT를 projected volume** 으로 마운트
6. 애플리케이션에서 JWT를 읽어 `/auth/kubernetes/login` 호출 → Vault 토큰 획득
7. 토큰으로 시크릿 경로에 `read` 요청 → 앱 내부에서 값 사용
8. 네트워크 캡처 및 Audit 로그로 전체 경로 시각화 및 검증

이 패턴을 이해하면 이후 **VSO(Vault Secrets Operator)**,
**Vault Agent Injector**, **External Secrets Operator** 등으로 확장할 때도
동일한 인증/인가 구조를 머릿속에 두고 비교할 수 있음

---

실습이 끝난 후에는 아래 명령으로 클러스터 정리:

```bash
kind delete cluster --name myk8s
```

(참고) K8S 1.34+ 에서는 파드 단에서 직접 인증서를 제공하는 **Pod Certificates** 기능이 알파/베타로 도입 중이며,
향후 Vault TLS 및 mTLS 연동과도 시너지를 낼 수 있음
