# 5주차 - Argo CD 2/3

## 목차

1. [전체 아키텍처 개요](#-전체-아키텍처)
2. [실습 구성 (Ingress-Nginx + ArgoCD TLS)](##-실습-구성--ingress-nginx--argocd-tls)

   - kind 클러스터 배포 개념 & 실습
   - Ingress-Nginx 설치 및 SSL Passthrough
   - ArgoCD with TLS (self-signed) 설치
3. [ArgoCD 접근 제어 개념](#-argocd-접근-제어-개념)

   - 계정 유형 (로컬 계정 / SSO 계정 / 서비스 어카운트)
   - RBAC 리소스와 액션
   - 최소 권한 원칙과 보안 패턴
4. [ArgoCD 접근 제어 실습](#-argocd-접근-제어-실습)

   - 선언적 로컬 사용자 관리 (alice)
   - RBAC 정책 설정 (argocd-rbac-cm)
   - CI/CD용 서비스 어카운트 생성 및 토큰 사용
5. [Keycloak 개념 정리](#-keycloak-개념-정리)

   - Keycloak이란?
   - 핵심 개념: Realm / Client / User / Group / Role
   - 주요 기능 및 표준 프로토콜
6. [Keycloak 설치 및 기본 구성 실습](#️-keycloak-설치-및-기본-구성-실습)

   - Docker 기반 Keycloak 실행
   - Realm 생성 (myrealm)
   - 사용자, 그룹, 역할 생성 및 매핑
   - Account Console 사용
7. [ArgoCD와 Keycloak SSO(OIDC) 연동](#-argocd와-keycloak-ssooidc-연동)

   - 전체 인증 흐름 이해 (Authorization Code Flow)
   - Keycloak Client(argocd) 생성
   - ArgoCD oidc.config 설정 및 Secret 연동
   - 그룹 기반 RBAC 연동 아이디어
8. [OAuth 2.0 & OpenID Connect 개념](#-oauth-20--openid-connect-개념)

   - OAuth 2.0 Authorization Code Flow
   - OAuth 2.0 vs OIDC 차이
   - Access Token / ID Token / Refresh Token 역할

---

## 전체 아키텍처

```text
[개발자 브라우저]
      │  HTTPS (https://argocd.example.com)
      ▼
[Ingress-Nginx Controller]  --(SSL Passthrough)-->  [ArgoCD Server Pod]
                                                      │
                                                      │ OIDC (Authorization Code)
                                                      ▼
                                         [Keycloak (IdP, Authorization Server)]
```

* **클러스터:** kind 기반 단일 노드 Kubernetes (control-plane 1개)
* **Ingress:** kind용 Ingress-Nginx 매니페스트 사용, `hostPort 80/443` 매핑
* **ArgoCD:** Helm으로 설치, 자체 TLS 사용 (argoCD 서버 Pod에서 TLS 종료)
* **Keycloak:** Docker 컨테이너로 별도 실행, OIDC Provider 역할
* **인증 흐름:** 브라우저 → ArgoCD → Keycloak → 토큰 발급 → ArgoCD 세션 생성

---

## 실습 구성 (Ingress-Nginx + ArgoCD TLS)

### kind 클러스터 배포

#### (1) 개념 정리

* **kind(Kubernetes IN Docker)**

  * Docker 컨테이너 하나를 K8s 노드처럼 사용하는 경량 클러스터 도구
  * 실습/테스트용으로 가볍고 재생성이 쉬움
* **extraPortMappings**

  * kind 노드(컨테이너)의 포트를 호스트로 노출
  * 여기서는 `80/443`(Ingress) + `30000~30003`(NodePort 테스트용)

#### (2) 실습 명령어

```bash
# kind 클러스터 생성
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
  - containerPort: 30002
    hostPort: 30002
  - containerPort: 30003
    hostPort: 30003
EOF

# 클러스터 상태 확인
kubectl cluster-info
kubectl get nodes
```

#### (3) kube-ops-view로 클러스터 시각화

```bash
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view \
  --version 1.2.2 \
  --set service.main.type=NodePort,service.main.ports.http.nodePort=30001 \
  --set env.TZ="Asia/Seoul" \
  --namespace kube-system

# 접속 URL
open "http://127.0.0.1:30001/#scale=1.5"
```

> **요약 포인트**
>
> * kind는 *실습용 K8s*를 빠르게 만들기 위한 도구
> * extraPortMappings로 **호스트 80/443 → Ingress**, **3000x → NodePort** 라우팅

---

### Ingress-Nginx 설치 및 설정

#### (1) 개념 정리

* **Ingress Controller**

  * `Ingress` 리소스를 감시하면서 L7(HTTP/HTTPS) 트래픽을 서비스로 라우팅
* **kind용 매니페스트 특징**

  * `hostPort: 80/443` 사용 → 호스트 포트 직접 바인딩
  * `nodeSelector: ingress-ready=true` → 특정 노드에만 배포
  * `tolerations`로 control-plane 노드 taint 허용

#### (2) 설치 및 확인

```bash
# 노드 라벨 확인
kubectl get nodes myk8s-control-plane -o jsonpath='{.metadata.labels}' | jq

# Ingress-Nginx 배포
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 리소스 확인
kubectl get deploy,svc,ep ingress-nginx-controller -n ingress-nginx
kubectl describe -n ingress-nginx deployments/ingress-nginx-controller
```

#### (3) SSL Passthrough 활성화

**왜 필요한가?**

* ArgoCD 서버 Pod가 **직접 TLS를 종료**하게 만들고 싶기 때문
* Ingress에서 TLS를 종료하지 않고, **암호화된 채로 그대로 전달** (L4에 더 가까운 동작)
* 잘못 설정하면 "리디렉션 횟수가 너무 많습니다" 같은 Redirect loop 발생

```bash
# 컨트롤러가 SSL 옵션을 지원하는지 확인
kubectl exec -it -n ingress-nginx deployments/ingress-nginx-controller -- \
  /nginx-ingress-controller --help | grep ssl

# Deployment 수정
KUBE_EDITOR="nano" kubectl edit -n ingress-nginx deployments/ingress-nginx-controller

# spec.template.spec.containers[].args 에 아래 추가
# - --enable-ssl-passthrough
```

#### (4) IPTABLES 규칙 확인 (네트워크 흐름 이해)

```bash
# control-plane 노드(컨테이너) 접속
docker exec -it myk8s-control-plane bash

iptables -t nat -L -n -v | grep '10.244.0.7'
exit

# 예시 출력
# DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:80  to:10.244.0.7:80
# DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:443 to:10.244.0.7:443
```

> **요약 포인트**
>
> * Ingress-Nginx는 **L7 라우터**지만, SSL Passthrough 시 사실상 **암호화된 TCP 스트림**을 그대로 전달
> * 이를 통해 ArgoCD 서버가 **자체 인증서**를 사용하도록 구성할 수 있음

---

### ArgoCD with TLS 설치

#### (1) self-signed TLS 인증서 생성

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout argocd.example.com.key \
  -out argocd.example.com.crt \
  -subj "/CN=argocd.example.com/O=argocd"

ls -l argocd.example.com.*
openssl x509 -noout -text -in argocd.example.com.crt
```

#### (2) TLS Secret 생성

```bash
kubectl create ns argocd

kubectl -n argocd create secret tls argocd-server-tls \
  --cert=argocd.example.com.crt \
  --key=argocd.example.com.key

kubectl get secret -n argocd argocd-server-tls
```

#### (3) Helm으로 ArgoCD 설치 (Ingress + TLS)

```bash
cat <<EOF > argocd-values.yaml
global:
  domain: argocd.example.com

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    tls: true
EOF

helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 9.0.5 \
  -f argocd-values.yaml \
  --namespace argocd

kubectl get pod,ingress,svc,ep,secret,cm -n argocd
kubectl get ingress -n argocd argocd-server
```

#### (4) 도메인 설정 및 접속 테스트

```bash
# /etc/hosts (macOS)
echo "127.0.0.1 argocd.example.com" | sudo tee -a /etc/hosts
cat /etc/hosts

# 접속 확인
curl -vk https://argocd.example.com/

# ArgoCD 서버/Ingress 로그 확인
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
kubectl -n argocd logs deploy/argocd-server

# 초기 admin 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# 브라우저 접속
open "https://argocd.example.com"
# admin / <위에서 조회한 비밀번호>
```

#### (5) ArgoCD CLI 로그인

```bash
argocd login argocd.example.com --insecure
# Username: admin
# Password: <초기 비밀번호>

argocd account list
argocd proj list
argocd repo list
argocd cluster list
argocd app list
```

> **요약 포인트**
>
> * ArgoCD 서버는 `argocd-server-tls` Secret에서 인증서를 로드
> * Ingress는 `ssl-passthrough`로 **암호화된 트래픽만 프록시**
> * 브라우저 ↔ ArgoCD Pod까지 **end-to-end HTTPS** 유지

---

## ArgoCD 접근 제어 개념

### 계정 유형

1. **로컬 계정 (local users)**

   * `argocd-cm`의 `accounts.<name>`로 정의
   * CLI / UI에 직접 로그인 가능 (`login` capability)
2. **SSO 계정 (OIDC/SAML)**

   * IdP(Keycloak)에서 인증 → ArgoCD는 토큰 기반으로 사용자 신원 인식
   * 보통 `groups` 또는 `email`을 RBAC와 매핑
3. **서비스 어카운트 (service accounts)**

   * 사람 대신 **CI/CD 시스템**이 사용하는 계정
   * UI 로그인 불필요, 주로 `apiKey`로 인증
   * 최소 권한으로 제한하는 것이 핵심

### RBAC 리소스와 동작

**정책 형식**

```text
p, <주체>, <리소스>, <동작>, <객체>, <효과>
g, <사용자 또는 그룹>, <역할>
```

* **주체(Subject)**: 사용자 이름, 그룹 이름, 또는 `role:<name>`
* **리소스(Resource)** 예시

  * `applications` / `projects` / `clusters` / `repositories` / `accounts` / `certificates` / `logs` / `exec` 등
* **동작(Action)**

  * `get`, `create`, `update`, `delete`, `sync`, `override`, `action/*` 등
* **객체(Object)**

  * `*/ *` : 모든 리소스
  * `default/*` : default namespace 애플리케이션
  * `default/myapp` : 특정 애플리케이션

### 최소 권한 원칙 (Least Privilege)

* `admin` 계정은 **비상용 + 초기 세팅용**으로만 사용하고 기본적으로 비활성화
* 팀 작업은 **개인 계정/SSO 계정**에 권한을 부여
* CI/CD는 **서비스 어카운트에 필요한 권한만 부여**
* `policy.default: role:readonly`와 같이 기본값을 읽기 전용으로 두고,
  필요한 사용자/그룹에만 추가 권한을 부여하는 패턴이 안전하다.

---

## ArgoCD 접근 제어 실습

### 선언적 로컬 사용자 관리 (alice)

#### (1) alice 계정 생성

```bash
KUBE_EDITOR="nano" kubectl edit cm -n argocd argocd-cm
```

`data` 섹션에 추가:

```yaml
accounts.alice: apiKey, login
```

> `apiKey`: API 토큰 발급 가능
> `login`: UI/CLI 로그인 허용

ConfigMap 적용 후 계정 확인:

```bash
argocd account list
# NAME    ENABLED  CAPABILITIES
# admin   false    login
# alice   true     apiKey, login
```

#### (2) alice 암호 설정 및 로그인

```bash
argocd account update-password --account alice
# Current password: <admin 암호>
# New password: <alice 암호>
# Confirm new password: <alice 암호>

argocd logout
argocd login argocd.example.com --username alice --insecure
```

---

### RBAC 정책 구성

```bash
KUBE_EDITOR="nano" kubectl edit cm -n argocd argocd-rbac-cm
```

예시 정책:

```yaml
data:
  policy.default: role:readonly
  policy.csv: |
    # alice에게 모든 애플리케이션 읽기 및 동기화 권한
    p, alice, applications, get, */*, allow
    p, alice, applications, sync, */*, allow

    # default 프로젝트 애플리케이션 전체 권한
    p, alice, applications, *, default/*, allow
```

ConfigMap 수정 후 ArgoCD 서버 재시작:

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

> **패턴 기억하기**
>
> * `policy.default`로 기본 권한 설정
> * `policy.csv`에서 사용자/그룹별 세부 권한 부여

---

### 서비스 어카운트 (gitops-ci)

#### (1) 서비스 어카운트 정의

```bash
KUBE_EDITOR="nano" kubectl edit cm -n argocd argocd-cm
```

`data` 섹션에 추가:

```yaml
accounts.gitops-ci: apiKey
```

계정 확인:

```bash
argocd account list
# NAME        ENABLED  CAPABILITIES
# admin       false    login
# alice       true     apiKey, login
# gitops-ci   true     apiKey
```

#### (2) API 토큰 생성 및 사용

```bash
argocd account generate-token -a gitops-ci
# → 이 토큰을 CI/CD 시스템의 Secret 등에 저장

export ARGOCD_AUTH_TOKEN=<생성된-토큰>
argocd app list --auth-token $ARGOCD_AUTH_TOKEN
```

#### (3) 서비스 어카운트 RBAC 제한

```bash
KUBE_EDITOR="nano" kubectl edit cm -n argocd argocd-rbac-cm
```

예시:

```yaml
policy.csv: |
  # gitops-ci는 default/myapp 애플리케이션만 제어
  p, gitops-ci, applications, get, default/myapp, allow
  p, gitops-ci, applications, sync, default/myapp, allow
  p, gitops-ci, applications, create, */*, deny
  p, gitops-ci, applications, delete, */*, deny
```

> **실무 팁**
>
> * CI 계정에는 `create/delete` 권한을 최소화하고, 주로 `sync`/`get` 위주로 부여
> * 토큰은 Git 저장소가 아니라 **CI 시스템의 Secret Backend**에 보관

---

## Keycloak 개념 정리

### Keycloak이란?

* 오픈 소스 **ID & Access Management** 솔루션
* 웹/모바일 애플리케이션에 **로그인·권한 관리** 기능을 외부 서비스로 분리해줌
* SSO, 소셜 로그인, LDAP/AD 연동, MFA, 세션 관리 등을 제공

### 핵심 개념 정리

* **Realm**

  * 독립적인 보안 영역 (Tenant 개념)
  * 각 Realm마다 **사용자, 클라이언트, 설정**이 분리
* **Client**

  * Keycloak이 보호하는 애플리케이션 (예: `argocd`, `grafana`)
  * OIDC/SAML Client 설정을 통해 리다이렉트 URL, Client Secret 등을 관리
* **User**

  * 실제 로그인하는 사용자 계정
* **Group**

  * 사용자 묶음, 공통 속성/권한/역할을 부여하기 좋음
* **Role**

  * 권한의 논리적 이름 (예: `admin`, `developer`)
  * Realm Role / Client Role로 나뉨
* **Identity Provider (IdP)**

  * Google, GitHub 등의 외부 로그인 제공자와 연동하는 브로커 기능
* **User Federation**

  * AD/LDAP 사용자 디렉토리를 그대로 사용

### 주요 기능 & 표준 프로토콜

* **기능**

  * 강력한 인증 (MFA, 패스워드 정책, 약관 동의 등)
  * Single Sign-On / Single Log-Out
  * 소셜 로그인 (Google, GitHub 등)
  * LDAP/AD 연동
  * 클러스터링 및 고가용성 지원
* **표준 프로토콜**

  * OAuth 2.0 (권한 부여)
  * OpenID Connect (인증 + 권한)
  * SAML 2.0 (엔터프라이즈 SSO)

---

## Keycloak 설치 및 기본 구성 실습

### Docker로 Keycloak 실행

```bash
# 기본 관리자 계정: admin / admin
docker run -d \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  --net host \
  --name dev-keycloak \
  quay.io/keycloak/keycloak:22.0.0 start-dev

# 컨테이너 확인
docker ps

# 로그 확인
docker logs dev-keycloak -f
```

접속:

```bash
open http://localhost:8080/admin
# admin / admin
```

---

### Realm 생성 (myrealm)

1. Admin Console 접속
2. 좌측 상단 Realm Selector → **Create Realm**
3. `Realm name: myrealm` → **Create**
4. `Realm settings → General`에서 Display name 세팅 가능

중요 Endpoint:

* OIDC Discovery: `http://localhost:8080/realms/myrealm/.well-known/openid-configuration`
* SAML Descriptor: `http://localhost:8080/realms/myrealm/protocol/saml/descriptor`

---

### 사용자 / 그룹 / 역할 관리

#### (1) User 생성

1. 왼쪽 메뉴 **Users** → **Add user**
2. 예시:

   * Username: `keycloak`
   * Email: `keycloak@keycloak.org`
   * First name: `Ola`
   * Last name: `Nordmann`
3. 생성 후 → **Credentials** 탭 → `Set password`

   * Password: 원하는 값
   * Temporary: `Off` (강제 변경 없음)

#### (2) Group 생성 및 사용자에 할당

1. **Groups → Create group**

   * Name: `mygroup`
2. Users → `keycloak` 선택 → **Groups** 탭 → `Join Group` → `mygroup` 선택

#### (3) Realm Role 생성 및 사용자에 매핑

1. **Realm roles → Create role**

   * Role name: `myrole`
2. Users → `keycloak` → **Role mapping** 탭 → `Assign role` → `myrole` 선택

---

### Account Console 사용

* URL 예: `http://localhost:8080/realms/myrealm/account`
* 사용자: `keycloak / <설정한 암호>`

주요 기능:

* 프로필 수정, 패스워드 변경
* 2단계 인증 설정
* 현재 로그인된 애플리케이션 보기
* 세션 강제 종료

---

## ArgoCD와 Keycloak SSO(OIDC) 연동

### 전체 인증 흐름 (Authorization Code Flow)

```text
[User] → [ArgoCD] → (리디렉트) → [Keycloak 로그인] → (코드) → [ArgoCD] → (토큰 교환) → 세션 생성
```

* ArgoCD는 **OIDC Client** 역할
* Keycloak은 **Authorization Server + IdP** 역할
* 사용자는 Keycloak 로그인 페이지에서 인증하고, ArgoCD는 토큰으로 사용자 정보를 얻는다.

---

### Keycloak Client(argocd) 생성

1. Admin Console → **Clients** → **Create client**
2. 설정:

   * Client type: `OpenID Connect`
   * Client ID: `argocd`
3. Capability config:

   * Client authentication: **On** (confidential client)
4. Login settings 예시:

   * Root URL: `https://argocd.example.com/`
   * Home URL: `/applications`
   * Valid redirect URIs: `https://argocd.example.com/auth/callback`
   * Valid post logout redirect URIs: `https://argocd.example.com/applications`

**Client Secret 확인**

* 생성된 `argocd` 클라이언트 → **Credentials** 탭 → Client secret 복사

---

### ArgoCD OIDC 설정

#### (1) Client Secret을 argocd-secret에 저장

```bash
kubectl -n argocd patch secret argocd-secret --patch='{"stringData": {
  "oidc.keycloak.clientSecret": "<Client-Secret-값>"
}}'

kubectl get secret -n argocd argocd-secret -o jsonpath='{.data}' | jq
```

#### (2) `argocd-cm`에 oidc.config 추가

> `issuer`의 IP는 **Keycloak이 띄워진 호스트의 실제 IP**로 변경할 것

```bash
# 로컬 IP 확인 (예: 192.168.254.110)
ifconfig | grep 192.

KUBE_EDITOR="nano" kubectl edit cm -n argocd argocd-cm
```

`data` 섹션에 예시로 추가:

```yaml
url: https://argocd.example.com

oidc.config: |
  name: Keycloak
  issuer: http://192.168.254.110:8080/realms/master
  clientID: argocd
  clientSecret: $oidc.keycloak.clientSecret
  requestedScopes: ["openid", "profile", "email", "groups"]
```

ArgoCD 서버 재시작:

```bash
kubectl rollout restart deploy argocd-server -n argocd
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server
```

#### (3) SSO 로그인 테스트

1. 브라우저에서 `https://argocd.example.com` 접속
2. `LOG IN VIA KEYCLOAK` 버튼 클릭
3. Keycloak 로그인 페이지에서 사용자(`keycloak / 비밀번호`) 입력
4. 다시 ArgoCD로 리디렉션 → 로그인 완료 확인

추가 사용자 예시:

* Username: `tom`
* Password: `tom123`

---

### 그룹 기반 RBAC 연동 아이디어

OIDC에서 `groups` claim을 포함하도록 설정하면, Keycloak 그룹을 ArgoCD RBAC에 그대로 매핑할 수 있다.

예시:

1. Keycloak에서 그룹 생성

   * `argocd-admins`, `frontend-team`, `backend-team` 등
2. 사용자를 각 그룹에 할당
3. ArgoCD `argocd-rbac-cm`에 다음과 같이 매핑

```yaml
policy.csv: |
  # Keycloak 그룹을 ArgoCD 역할에 연결
  g, argocd-admins, role:admin

  # frontend-team은 frontend-* 애플리케이션만 관리
  p, frontend-team, applications, *, frontend-*, allow

  # backend-team은 backend-* 애플리케이션만 관리
  p, backend-team, applications, *, backend-*, allow
```

이렇게 하면 **그룹만 관리해도 권한이 자동으로 반영**되므로, 대규모 조직에서 매우 유용하다.

---

## OAuth 2.0 & OpenID Connect 개념

### OAuth 2.0 Authorization Code Flow

**역할**

* Resource Owner: 사용자
* Client: 애플리케이션 (ArgoCD)
* Authorization Server: Keycloak
* Resource Server: API 서버 (예: Git 서비스, Protected API)

**일반적인 흐름**

1. 클라이언트 애플리케이션이 "보호된 리소스"를 사용하려 함
2. Authorization Server로 리디렉트 → 사용자 로그인 및 권한 동의
3. Authorization Server가 **Authorization Code** 발급
4. 클라이언트가 코드를 이용해 **Access Token**(+ 필요 시 Refresh Token)을 요청
5. Access Token을 들고 Resource Server에 API 요청

> 포인트: **사용자 자격증명은 Authorization Server에서만 처리**되고,
> 클라이언트는 Access Token만 사용한다.

---

### 2️⃣ OAuth 2.0 vs OpenID Connect

| 구분    | OAuth 2.0             | OpenID Connect              |
| ----- | --------------------- | --------------------------- |
| 목적    | 권한 부여 (Authorization) | 인증(Authentication) + 권한 부여  |
| 토큰    | Access Token          | Access Token + **ID Token** |
| 사용 사례 | API 접근 제어             | 로그인/SSO, 사용자 신원 확인          |
| 표준    | RFC 6749 등            | OpenID Connect Core 1.0     |

OIDC는 OAuth 2.0 위에 **사용자 신원 정보를 표준화**해서 올려놓은 스펙이라고 이해하면 된다.

---

### 토큰 종류와 역할

#### (1) Access Token

* 목적: API 리소스 호출 권한 부여
* 특징:

  * 보통 수 분~수십 분 정도의 짧은 수명
  * Resource Server(Nginx, API Gateway 등)가 검증
  * Scope 기반으로 권한을 제한

#### (2) ID Token (OIDC)

* 목적: 사용자 신원 정보 전달
* 형식: 대개 JWT(JSON Web Token)

예시 Claim:

```json
{
  "iss": "http://localhost:8080/realms/master",
  "sub": "user-id",
  "aud": "argocd",
  "exp": 1699999999,
  "email": "keycloak@keycloak.org",
  "preferred_username": "keycloak",
  "groups": ["mygroup"]
}
```

* `iss`: 토큰 발급자 (Keycloak Realm)
* `aud`: 토큰 대상 (Client ID)
* `sub`: 사용자 고유 ID
* `groups`: 사용자 그룹 정보 (RBAC에 활용 가능)

#### (3) Refresh Token

* 목적: Access Token 만료 시 새 토큰을 발급받기 위한 장기 토큰
* 특징:

  * 상대적으로 긴 수명 (수일~수개월)
  * 노출 시 위험도가 크므로 안전한 저장 필요

---

## 실무 적용 시나리오

### 시나리오 1: 엔터프라이즈 SSO 통합

**과제**: 회사의 AD/LDAP 사용자로 ArgoCD에 로그인하고 싶다.

* Keycloak User Federation으로 AD/LDAP 연동
* 그룹/OU를 ArgoCD RBAC에 매핑
* 내부 직원은 LDAP, 외부 협력사는 GitHub/Google OAuth 등으로 분기 가능
* Keycloak Events로 감사 로그(로그인/로그아웃) 추적

---

### 시나리오 2: 팀별 권한 분리

**과제**: Frontend/Backend/DevOps 팀의 권한을 분리하고 싶다.

* Keycloak 그룹

  * `frontend-team`, `backend-team`, `devops-team`
* ArgoCD RBAC 예시

```yaml
policy.csv: |
  # Frontend 팀
  p, frontend-team, applications, *, frontend-*, allow

  # Backend 팀
  p, backend-team, applications, *, backend-*, allow

  # DevOps 팀 전체 관리자
  g, devops-team, role:admin
```

사용자를 그룹에 추가/제거하는 것만으로 **권한이 자동 관리**된다.

---

### 시나리오 3: CI/CD 파이프라인 자동화

**과제**: Jenkins/GitLab CI에서 ArgoCD API를 사용해 애플리케이션을 Sync

* `accounts.ci-pipeline: apiKey` 서비스 어카운트 생성
* 최소 권한 부여

```yaml
p, ci-pipeline, applications, get, default/*, allow
p, ci-pipeline, applications, sync, default/*, allow
```

* API 토큰 발급 후 CI Secret에 저장

```bash
TOKEN=$(argocd account generate-token -a ci-pipeline)
argocd app sync myapp --auth-token $TOKEN
```

---

