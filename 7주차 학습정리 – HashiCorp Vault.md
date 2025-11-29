# 7주차 – HashiCorp Vault

## 목차 (Table of Contents)

1. [왜 HashiCorp Vault인가?](#1-왜-hashicorp-vault인가)

   * [우리가 다루는 "시크릿(Secret)"이란?](#우리가-다루는-시크릿secret이란)
   * [아키텍처 진화와 Secret Sprawl](#아키텍처-진화와-secret-sprawl)
   * [Zero Trust + Secret Management](#zero-trust--secret-management)
   * [우리의 목표](#우리의-목표)
2. [Vault Dev 모드 설치 + UI/CLI 접속](#2-vault-dev-모드-설치--uicli-접속)

   * [Dev Mode vs Production Mode](#개념--dev-mode-vs-production-mode)
   * [kind + Dev Mode Vault 설치](#실습--kind--dev-mode-vault-설치)
   * [UI / CLI 접속](#ui--cli-접속)
   * [체크 포인트](#체크-포인트)
3. [KV 시크릿 엔진 + Policy + AppRole](#3-kv-시크릿-엔진--policy--approle)

   * [Secrets Engine 개념](#개념)
   * [Policy 설정](#policy--경로path-기반-접근-제어)
   * [AppRole 인증](#approle--머신machine용-계정)
   * [실습 및 체크 포인트](#실습)
4. [Vault Agent + Sidecar 패턴 (Kubernetes)](#4-vault-agent--sidecar-패턴-kubernetes)

   * [Vault Agent의 역할](#vault-agent의-역할)
   * [Kubernetes Vault Agent Injector](#kubernetes-vault-agent-injector)
   * [실습 및 배포 예제](#실습--kubernetes-auth--sidecar-주입)
   * [체크 포인트](#체크-포인트-1)
5. [Transit 엔진 – Encryption as a Service](#5-transit-엔진--encryption-as-a-service)

   * [Transit 개념 및 구조](#개념-1)
   * [실습 – 암복호화](#실습--transit-엔진-활성화-및-암복호화)
   * [체크 포인트](#체크-포인트-2)
6. [Jenkins + Vault – CI 파이프라인 시크릿 관리](#6-jenkins--vault--ci-파이프라인-시크릿-관리-개념-위주)

   * [하드코딩의 위험성](#jenkins에-하드코딩된-시크릿의-위험)
   * [AppRole 및 DB 연동 개요](#vault-측-설정-요약-jenkins-approle)
   * [체크 포인트](#체크-포인트-3)
7. [ArgoCD + Vault Plugin – GitOps 시크릿 통합](#7-argocd--vault-plugin--gitops와-시크릿-통합)

   * [GitOps 딜레마와 해결 방식](#gitops의-딜레마)
   * [AppRole 설정 및 Placeholder](#vault-policy--approle-argocd용)
   * [체크 포인트](#체크-포인트-4)
8. [Vault Secrets Operator (VSO) – Vault → K8s Secret 동기화](#8-vault-secrets-operator-vso--vault--kubernetes-secret-동기화)

   * [VSO 목적 및 CRD 구성](#vso의-목적)
   * [실습 – Secret 동기화 예제](#실습--정적-시크릿-동기화)
   * [체크 포인트](#체크-포인트-5)


---

## 1. 왜 HashiCorp Vault인가?

### 우리가 다루는 “시크릿(Secret)”이란?

노출되면 큰 피해가 발생하는 모든 값들:

* DB 계정(ID/PW)
* 클라우드 크리덴셜(AWS Access Key, GCP SA Key 등)
* API Key / Token (GitHub, Slack, OpenAI 등)
* SSH Key, TLS 개인키(Private Key)

**문제점 (안티패턴)**

```yaml
# 소스코드 / YAML / Jenkinsfile에 박아 넣는 방식
DB_PASSWORD: "admin123"
AWS_SECRET_KEY: "AKIA..."
```

* Git 히스토리에 평생 남음
* 실수로 Public Repo에 push → 즉시 유출
* 로테이션(비밀번호 변경) 어려움
* 누가 어디서 쓰는지 파악 불가(Secret Sprawl)

---

### 아키텍처 진화와 Secret Sprawl

1. **모놀리식 / 온프레미스 시대**

* 서버 수 적음, 애플리케이션 수 적음
* 시크릿도 몇 개 안 됨 → `/etc/app.conf`에 넣어도 어떻게든 관리

2. **MSA + 클라우드 + K8s 시대**

* 마이크로서비스 수십~수백, Pod는 분 단위로 생성/삭제
* 사람보다 **머신 계정**(서비스계정, CI/CD 파이프라인)이 더 많음
* 각 서비스마다 DB 계정, API Key, Token이 개별 관리 → Secret Sprawl 심각

Secret Sprawl: (API 키, 비밀번호, 암호화 키 등 민감한 정보가 조직 내에서 통제되지 않고 여러 곳에 산발적으로 흩어지는 현상)
---

### Zero Trust + Secret Management

**Zero Trust 기본 원칙**

* "내부니까 안전하겠지" No
* 모든 요청에 대해:

  1. **너 누구니?** (Authentication)
  2. **뭘 할 수 있나?** (Authorization)
  3. **언제 뭘 했는가?** (Audit)

Vault는 이걸 **시크릿 관점**에서 구현:

* 이 토큰/역할은 `어떤 경로(path)의 시크릿`만 접근 가능
* DB 계정을 **요청할 때마다 동적으로 생성**, TTL 끝나면 자동 삭제
* 누가 언제 어떤 시크릿에 접근했는지 **Audit 로그**로 남김

---

### 우리의 목표

* Kubernetes에 Vault를 배포하고 (Dev → Prod 개념 이해)
* KV / Database / Transit 엔진을 실습으로 익히고
* Vault Agent, Jenkins, ArgoCD Vault Plugin, Vault Secrets Operator(VSO)로
  **CI/CD와 GitOps 파이프라인에 Vault를 통합**해 보는 것

즉, “GitOps + CI/CD + Vault” 조합으로 **프로덕션급 시크릿 파이프라인**을 훑어 본다.

---

## 1. Vault Dev 모드 설치 + UI/CLI 접속

### 개념 – Dev Mode vs Production Mode

**Dev Mode**

* In-Memory Storage (메모리에 저장 → Pod 재시작 시 초기화)
* Unseal 과정 없음 (자동 Unseal)
* `devRootToken`으로 Root 토큰 고정 가능
* 목적: **로컬 실습, PoC, 명령어 익히기**

**Production Mode**

* 실제 스토리지(Raft, Consul 등)에 암호화 저장
* 시작 시 항상 **Sealed 상태** → 여러 개의 Unseal Key를 모아야 Open
* TLS, Auto-Unseal(KMS, 다른 Vault Transit 등) 필수

---

### 실습 – kind + Dev Mode Vault 설치

#### kind 클러스터 생성

```bash
kind create cluster --name vault-demo --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
EOF

kubectl cluster-info
kubectl get nodes
```

#### Helm으로 Vault Dev 설치

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace vault

helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=root" \
  --set "ui.enabled=true" \
  --set "ui.serviceType=NodePort" \
  --set "ui.serviceNodePort=30000"

kubectl get pod,svc -n vault
```

#### UI / CLI 접속

* UI: 브라우저에서 `http://127.0.0.1:30000`

  * **Method**: Token
  * **Token**: `root`

* CLI 환경변수 설정:

```bash
export VAULT_ADDR='http://127.0.0.1:30000'
export VAULT_TOKEN='root'

vault status
vault login $VAULT_TOKEN
vault version
```

### 체크 포인트

* Dev 모드는 **연습용** → 실제 운영에 사용하면 안 됨.
* UI와 CLI 둘 다 **같은 Vault 인스턴스**를 보는 것임을 이해하기.
* `VAULT_ADDR`, `VAULT_TOKEN`이 잡혀 있지 않으면 `permission denied` 발생.

---

## 2. KV 시크릿 엔진 + Policy + AppRole

### 개념

#### Secrets Engine이란?

* Vault 코어는 요청을 라우팅하고, 실제 시크릿 저장/생성은 **엔진** 단위로 수행
* 주요 엔진 예시:

  * **KV 엔진**: Key-Value 정적 시크릿 (config처럼 쓰는 상수)
  * **Database 엔진**: DB 계정을 **요청 시마다 동적으로 생성/폐기**
  * **Transit 엔진**: 암호화/복호화를 대신 수행 (Encryption as a Service)

#### (2) Policy – 경로(Path) 기반 접근 제어

* Vault는 경로(`path`) 단위로 권한을 정의:

```hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
```

* Policy = 이 토큰/역할이 **어떤 path에 어떤 권한(capabilities)**을 가지는지 정의
* capability 종류: `read`, `list`, `create`, `update`, `delete`, `sudo` 등

#### AppRole – 머신(Machine)용 계정

* 사람은 GitHub, LDAP, Token 등으로 로그인
* Jenkins, Cron Job, 애플리케이션은 **AppRole**로 인증
* AppRole = `RoleID + SecretID`

  * RoleID: 역할의 ID (공개 가능)
  * SecretID: Role에 대한 비밀키 (토큰(Access Token)을 발급받기 위한 자격)

---

### 실습

#### Vault Pod 접속

```bash
kubectl exec -it -n vault vault-0 -- sh

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'
```

#### KV v2 엔진 활성화 + 시크릿 저장

```bash
# KV v2 엔진 활성화
vault secrets enable -path=secret kv-v2

# 시크릿 저장
vault kv put secret/myapp/config \
  username='admin' \
  password='secret123'

# 시크릿 조회
vault kv get secret/myapp/config
```

* kv-v2 구조상, 실제 API 경로는 `secret/data/myapp/config` 형태를 사용.

#### Policy 생성

```bash
vault policy write myapp-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF
```

#### AppRole 생성

```bash
vault auth enable approle

vault write auth/approle/role/myapp \
  token_policies="myapp-policy" \
  token_ttl=1h \
  token_max_ttl=4h

# RoleID / SecretID 확인
vault read auth/approle/role/myapp/role-id
vault write -f auth/approle/role/myapp/secret-id
```

* 출력된 `role_id`, `secret_id`는 CI/CD, 애플리케이션 설정에 사용

### 체크 포인트

* `kv-v2`는 항상 `/data/` 경로가 추가된다는 점(`secret/data/...`) 기억.
* Policy는 **최소 권한 원칙**으로 작성 (필요한 경로만 허용).
* AppRole은 “머신 계정” → SecretID는 다른 시크릿처럼 안전하게 보관.

---

## 3. Vault Agent + Sidecar 패턴 (Kubernetes)

### 개념

#### Direct Integration의 문제점

* 애플리케이션이 Vault SDK를 직접 사용하면:

  * 언어별 구현(Java, Go, Python, Node.js 등) 모두 따로 관리
  * 토큰 갱신/Lease 관리/에러 처리/재시도 로직 등 추가 구현 필요
  * 비즈니스 로직 + 보안/인프라 로직이 뒤섞임

#### Vault Agent의 역할

* Vault Agent가 대신:

  * 자동 인증 (Kubernetes Service Account, AppRole, AWS IAM, GCP SA 등)
  * 시크릿 요청 및 템플릿 렌더링 → 파일에 저장
  * 토큰/Lease 자동 갱신 (TTL 만료 전 갱신)
  * 결과 캐싱 (Vault API 호출 감소)

* 애플리케이션 코드는 **“파일 읽기”만 하면 됨**

#### Kubernetes Vault Agent Injector

* Mutating Admission Webhook 기반으로 Pod 생성 시점에 **자동으로**:

  * Init Container (`vault-agent-init`) 추가
  * Sidecar Container (`vault-agent`) 추가
* Pod 템플릿에 annotation만 추가하면 동작

---

### 실습 – Kubernetes Auth + Sidecar 주입

#### Kubernetes Auth 활성화

```bash
kubectl exec -it -n vault vault-0 -- sh

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

#### Policy & Role 설정

```bash
vault policy write myapp-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=default \
  policies=myapp-policy \
  ttl=1h
```

#### 테스트용 시크릿 저장

```bash
vault kv put secret/myapp/config \
  username='admin' \
  password='secret123'
```

#### Deployment 작성 (Sidecar 자동 주입)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"

        # Vault에서 읽어올 시크릿 경로
        vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"

        # 템플릿: 파일 내용 정의
        vault.hashicorp.com/agent-inject-template-config.txt: |
          USERNAME={{- with secret "secret/data/myapp/config" -}}{{ .Data.data.username }}{{- end }}
          PASSWORD={{- with secret "secret/data/myapp/config" -}}{{ .Data.data.password }}{{- end }}
    spec:
      serviceAccountName: myapp
      containers:
      - name: myapp
        image: nginx:latest
        command:
        - sh
        - -c
        - |
          echo "Reading secrets from /vault/secrets/config.txt"
          cat /vault/secrets/config.txt
          sleep 3600
```

배포 및 확인:

```bash
kubectl apply -f myapp-deployment.yaml

kubectl get pod
kubectl describe pod myapp-xxxxx  # init/sidecar 컨테이너 확인

kubectl exec -it <pod-name> -c myapp -- cat /vault/secrets/config.txt
# USERNAME=admin
# PASSWORD=secret123
```

### 체크 포인트

* annotation 오타 시 Injector가 동작하지 않음 → Key 이름/문자열 정확히.
* 템플릿에서 `.Data.data.username` 같이 **KV v2 구조**를 정확히 써야 함.
* Pod 안에는 **애플리케이션 컨테이너 + Init + Agent Sidecar** 3개가 생김.

---

## 4. Transit 엔진 – Encryption as a Service

### 개념

#### 기존 앱 내 암호화의 문제

* 앱 내부에서 AES 라이브러리를 직접 사용하면:

  * 키를 어디에 저장? (환경변수? 파일? 코드?)
  * 누가 키를 볼 수 있음?
  * 키를 로테이션하면 기존 데이터는 어떻게 복호화?

#### Transit 엔진의 역할

* **키는 Vault가 소유 및 관리**
* 애플리케이션은 `plaintext → encrypt → ciphertext`, `ciphertext → decrypt → plaintext` 요청만 수행
* 특징:

  * Key rotation 지원 (버전 관리)
  * 앱 코드는 키를 보지 못함 → 키 노출 위험 감소
  * API 호출 기반이므로 멀티 언어/멀티 플랫폼 공통 사용

#### 암호화 계층

* 네트워크(TLS), 디스크(LUKS) 암호화 외에
* **애플리케이션 계층**에서 민감 컬럼(이메일, 카드번호 등)을 Transit으로 암호화

---

### 실습 – Transit 엔진 활성화 및 암복호화

#### Transit 엔진 활성화

```bash
kubectl exec -it -n vault vault-0 -- sh
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault secrets enable transit
vault write -f transit/keys/my-key
vault read transit/keys/my-key
```

#### 암호화

```bash
PLAINTEXT_BASE64=$(echo -n "my secret data" | base64)

echo $PLAINTEXT_BASE64
# bXkgc2VjcmV0IGRhdGE=

vault write transit/encrypt/my-key plaintext="$PLAINTEXT_BASE64"
# 결과 예시:
# ciphertext    vault:v1:8SDd3WHD...
```

#### 복호화

```bash
CIPHERTEXT="vault:v1:8SDd3WHD..."

vault write transit/decrypt/my-key ciphertext="$CIPHERTEXT"
# plaintext: bXkgc2VjcmV0IGRhdGE=

echo "bXkgc2VjcmV0IGRhdGE=" | base64 -d
# my secret data
```

#### 키 Rotation

```bash
vault write -f transit/keys/my-key/rotate
vault read transit/keys/my-key   # latest_version 증가 확인
```

* 새로 암호화할 때는 최신 버전의 키 사용
* 예전 버전으로 암호화된 데이터도 (키를 파괴하지 않았다면) 복호화 가능

### 체크 포인트

* Transit API는 `plaintext`를 **Base64 인코딩 문자열로 받는다**는 점 주의.
* 애플리케이션 관점에서:

  * INSERT: `평문 → base64 → encrypt → ciphertext를 DB에 저장`
  * SELECT: `DB의 ciphertext → decrypt → base64 decode → 평문`
* Key rotation은 **암호화 정책 강화를 위해 주기적으로 해야 하는 작업**

---

## 5. Jenkins + Vault – CI 파이프라인 시크릿 관리 (개념 위주)

### 개념

#### Jenkins에 하드코딩된 시크릿의 위험

```groovy
// 안티 패턴
environment {
  DB_PASS      = "admin123"
  AWS_SECRET   = "AKIA..."
  SLACK_TOKEN  = "xoxb-..."
}
```

* Jenkinsfile도 Git에 저장되므로 시크릿이 남음
* Job 로그에 시크릿이 출력될 수도 있음

**목표:**

* Jenkins는 빌드 시점에 Vault에서 시크릿을 가져와 **환경 변수로만 일시 사용**
* 가능한 경우, DB 계정도 **동적(Dynamic)**으로 발급받아서 TTL 후 자동 폐기

#### KV vs Dynamic Secrets

* **KV(정적)**: 고정된 값 (예: 특정 API Key, Webhook URL)
* **Dynamic(Database)**: 요청 시마다 임시 계정 생성

  * 예: `vault-jenkins-xxxxx` 계정, TTL 10분
  * 유출되더라도 시간 지나면 자동 폐기 → 리스크 크게 감소

---

### Vault 측 설정 요약 (Jenkins AppRole)

```bash
vault policy write jenkins-policy - <<EOF
path "secret/data/jenkins/*" {
  capabilities = ["read"]
}
EOF

vault auth enable approle

vault write auth/approle/role/jenkins \
  token_policies="jenkins-policy" \
  token_ttl=1h \
  token_max_ttl=4h

vault read auth/approle/role/jenkins/role-id
vault write -f auth/approle/role/jenkins/secret-id
```

### Vault Database 엔진 (동적 Credential 개념)

```bash
vault secrets enable database

vault write database/config/mysql \
  plugin_name=mysql-database-plugin \
  connection_url=":@tcp(mysql.default:3306)/" \
  allowed_roles="jenkins-role" \
  username="root" \
  password="rootpassword"

vault write database/roles/jenkins-role \
  db_name=mysql \
  creation_statements="CREATE USER ''@'%' IDENTIFIED BY '';GRANT SELECT ON mydb.* TO ''@'%';" \
  default_ttl="10m" \
  max_ttl="1h"
```

Jenkins에서는 Vault Plugin의 `withVault { ... }` 블록에서
`database/creds/jenkins-role`을 읽어 `DB_USERNAME`, `DB_PASSWORD`를 환경 변수로 사용.

### 체크 포인트

* CI 파이프라인에서 Vault를 붙이는 목적:

  * 시크릿의 **유효 시간(TTL)**을 짧게 가져가고,
  * 하드코딩/로컬 파일/환경 변수 남용을 줄이는 것.
* 로그에 시크릿이 노출되지 않도록 `echo` 사용에 주의해야 함.

---

## 6. ArgoCD + Vault Plugin – GitOps와 시크릿 통합

### 개념

#### GitOps의 딜레마

* GitOps 철학: 모든(거의 모든) 설정을 Git에 선언적으로 저장
* 문제: Secret 리소스에 평문으로 ID/PW를 저장할 수 없음

기존 해결책들:

* Sealed Secrets, External Secrets Operator, SOPS 등

#### ArgoCD Vault Plugin의 접근 방식

* Git에는 **Placeholder**만 기록한다:

```yaml
env:
- name: DB_USERNAME
  value: <path:secret/data/myapp/config#username>
- name: DB_PASSWORD
  value: <path:secret/data/myapp/config#password>
```

* ArgoCD RepoServer(또는 CMP)에서 Sync 시:

  1. Manifest를 읽고
  2. `<path:...#key>` 패턴을 찾고
  3. Vault에서 해당 경로의 시크릿을 읽어서
  4. 실제 값으로 치환한 뒤 Kubernetes에 apply

→ Git에는 실제 시크릿 값이 절대 저장되지 않음.

---

### Vault Policy + AppRole (ArgoCD용)

```bash
vault policy write argocd-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

vault auth enable approle

vault write auth/approle/role/argocd \
  token_policies="argocd-policy" \
  token_ttl=1h \
  token_max_ttl=4h

vault read auth/approle/role/argocd/role-id
vault write -f auth/approle/role/argocd/secret-id
```

ArgoCD 쪽에서는 이 `ROLE_ID`, `SECRET_ID`를 Secret으로 만들어 환경 변수로 사용.

### Placeholder 문법 정리

```yaml
env:
- name: DB_USERNAME
  value: <path:secret/data/myapp/config#username>
#        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 경로
#                                              ^^^^^ 필드명
```

* `path:` 뒤: Vault에서 읽을 경로
* `#` 뒤: JSON/YAML 필드 이름

### 체크 포인트

* 이 방식은 **CD(ArgoCD) 단계에서만** 동작 (CI와 분리됨).
* Git에는 Placeholder만 있기 때문에 Repo가 노출되어도 시크릿 평문은 노출되지 않음.
* AppRole Policy를 최소 권한으로 작성해야 ArgoCD가 필요한 시크릿만 읽을 수 있음.

---

## 7. Vault Secrets Operator (VSO) – Vault → Kubernetes Secret 동기화

### 개념

#### VSO의 목적

* 애플리케이션/개발팀 입장:

  * 기존 방식 (`envFrom.secretKeyRef`, `volume.secret`) 그대로 쓰고 싶다.
* 플랫폼/보안팀 입장:

  * 시크릿의 근원은 Vault에 두고, 거기서만 관리하고 싶다.

Vault Secrets Operator(VSO)는:

* Vault에서 시크릿을 읽어와서
* Kubernetes Secret으로 생성/갱신해 주는 **Operator(컨트롤러)**

#### 주요 CRD

* `VaultConnection`: Vault 서버 연결 정보 (address, TLS 등)
* `VaultAuth`: Vault 인증 방식 (kubernetes, approle 등)
* `VaultStaticSecret`: KV처럼 정적 시크릿을 Secret으로 동기화
* `VaultDynamicSecret`: Database/AWS 등 동적 시크릿을 Secret으로 동기화

---

### 실습 – 정적 시크릿 동기화

#### VSO 설치

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  --namespace vault-secrets-operator-system \
  --create-namespace

kubectl get pod -n vault-secrets-operator-system
```

#### VaultConnection CRD

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: default
spec:
  address: http://vault.vault:8200
  skipTLSVerify: true
```

#### (3) VaultAuth (Kubernetes Auth)

Vault 설정:

```bash
vault policy write vso-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

vault auth enable kubernetes  # 이미 되어 있다면 생략

vault write auth/kubernetes/role/vso \
  bound_service_account_names=vso \
  bound_service_account_namespaces=default \
  policies=vso-policy \
  ttl=1h
```

Kubernetes 리소스:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vso
  namespace: default
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: default
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: vso
    serviceAccount: vso
```

#### VaultStaticSecret – Vault → K8s Secret

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: myapp-config
  namespace: default
spec:
  vaultAuthRef: vault-auth
  mount: secret
  type: kv-v2
  path: myapp/config
  destination:
    name: myapp-config-secret
    create: true
  refreshAfter: 30s
```

* Vault의 `secret/myapp/config` 내용을
* `myapp-config-secret`이라는 Kubernetes Secret으로 생성/동기화
* Vault 값 변경 시, 최대 30초 내에 Secret도 갱신

#### Pod에서 Secret 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: default
spec:
  serviceAccountName: vso
  containers:
  - name: myapp
    image: nginx:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: myapp-config-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: myapp-config-secret
          key: password
```

애플리케이션 입장에서는 **그냥 기존처럼 Secret을 읽고 있을 뿐**이지만,
실제로는 백그라운드에서 VSO가 Vault와 통신하여 값을 가져와 주는 구조.

### 체크 포인트

* 장점:

  * 앱/팀은 기존 K8s Secret 패턴을 그대로 사용 (변경 최소화)
  * Root of Truth(진짜 근원)는 Vault에 있기 때문에 로테이션, 감사, 권한 관리를 중앙에서 수행
* 단점:

  * etcd에 저장된 Secret은 여전히 별도 보호 필요 → K8s **Encryption at Rest** 설정과 함께 써야 완전
