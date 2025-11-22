# 6주차 - Argo CD 3/3

## 목차

1. [Cluster Management — kind + Ingress + Argo CD](#1-cluster-management--kind--ingress--argo-cd)
2. [dev / prd 클러스터 설치 및 kubeconfig 조정](#2-dev--prd-클러스터-설치-및-kubeconfig-조정)
3. [Helm Chart 제작 및 Multi-Cluster 배포](#3-helm-chart-제작-및-multi-cluster-배포)
4. [Argo CD ApplicationSet](#4-argo-cd-applicationset)
5. [Keycloak + Argo CD + Jenkins (OIDC SSO)](#5-keycloak--argo-cd--jenkins-oidc-sso)
6. [OpenLDAP + Keycloak 통합](#6-openldap--keycloak-통합)
7. [Argo CD RBAC 적용](#7-argo-cd-rbac-적용)
8. [클러스터 삭제](#8-클러스터-삭제)

---

# 1. Cluster Management — kind + Ingress + Argo CD

### ▶ 실습 체크리스트

1. mgmt 클러스터 생성(kind)
2. Ingress-NGINX 설치
3. SSL Passthrough 활성화
4. Argo CD 설치 및 초기 패스워드 확인
5. 브라우저 접속 확인

## 1.1 실습 개요

이 섹션은 **GitOps 컨트롤 플레인 구조를 이해하는 핵심 단계**입니다. kind 클러스터를 3개(mgmt/dev/prd)로 구성하여 실제 엔터프라이즈 환경의 멀티 클러스터 운영 방식을 재현합니다.

* **mgmt 클러스터**는 Argo CD, Ingress, SSO(Keycloak), CI(Jenkins) 등을 모두 탑재하는 중앙 제어 클러스터입니다.
* **dev/prd 클러스터**는 애플리케이션 배포 타깃이며, Argo CD가 GitOps 방식으로 관리합니다.
* 모든 클러스터는 Docker bridge 네트워크(kind)를 통해 서로 통신하며, mgmt에서 dev/prd로 API 요청을 전달하는 구조입니다.

이 설계는 다음과 같은 엔터프라이즈 GitOps 구조를 그대로 재현한 것입니다.

* Git → Argo CD(mgmt) → 여러 Kubernetes Cluster(dev/prd)

* 하나의 mgmt에서 수십~수백 클러스터를 제어할 수 있는 확장형 패턴입니다.
  Kind 클러스터 3개(mgmt/dev/prd)를 만들어 mgmt 클러스터에 Argo CD를 설치하고, dev/prd 클러스터를 GitOps로 관리하는 구조입니다.

* **mgmt:** Argo CD + Ingress-NGINX 설치, 모든 제어의 중심

* **dev/prd:** 애플리케이션 배포 대상 클러스터

* **통신:** Docker 네트워크(kind) 내부 IP 기반

## 1.2 실습 — kind mgmt 배포

```
kind create cluster --name mgmt --image kindest/node:v1.32.8 --config - <<EOF
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
```

## 1.3 Ingress 설치

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

SSL Passthrough 활성화:

```
kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml \
| sed '/- --publish-status-address=localhost/a\
        - --enable-ssl-passthrough' | kubectl apply -f -
```

## 1.4 Argo CD 설치

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout argocd.example.com.key \
  -out argocd.example.com.crt \
  -subj "/CN=argocd.example.com/O=argocd"
```

```bash
kubectl -n argocd create secret tls argocd-server-tls \
  --cert=argocd.example.com.crt \
  --key=argocd.example.com.key
```

로그인, 패스워드 변경 등 모든 명령어 그대로 유지.

---

# 2. dev / prd 클러스터 설치 및 kubeconfig 조정

### ▶ 실습 체크리스트

1. dev/prd 클러스터 각각 생성
2. Docker 네트워크 확인 (kind bridge)
3. dev/prd kubeconfig server IP 수정
4. Argo CD에 dev/prd 클러스터 등록
5. 정상 연결 확인 (argocd cluster list)

## 실습 개요

이 섹션은 **멀티 클러스터 GitOps 운영의 핵심 개념**을 다룹니다.

kind는 기본적으로 kubeconfig에 API 서버를 `127.0.0.1:<port>` 형태로 기록합니다. 이것은 **로컬 PC에서 직접 dev/prd 클러스터에 접근할 때**는 정상적으로 동작하지만, mgmt 클러스터 내부의 Argo CD가 dev/prd API 서버를 호출할 때는 문제가 됩니다.

Argo CD는 mgmt 클러스터의 Pod 내부에서 실행되기 때문에 dev/prd API 서버에 접근하려면 **Docker 네트워크(kind bridge)의 실제 IP(172.20.x.x:6443)** 를 사용해야 합니다.

즉,

* 로컬 kubectl → 127.0.0.1:port 접근
* Argo CD Pod → 반드시 172.20.0.x:6443 로 접근

따라서 dev/prd kubeconfig의 `clusters.cluster.server` 값을 **직접 수정해야만** Argo CD가 정상적인 멀티 클러스터 Sync를 수행할 수 있습니다.

## 실습 개요

* kind 기본 kubeconfig는 `127.0.0.1` 로 된 server 주소를 포함
* mgmt → dev/prd 간 호출 위해선 **Docker 네트워크 IP**를 명시해야 함

## 실습

```
kind create cluster --name dev --image kindest/node:v1.32.8 --config - <<EOF
...
EOF

kind create cluster --name prd --image kindest/node:v1.32.8 --config - <<EOF
...
EOF
```

Docker network inspect, kubeconfig 수정, cluster add 명령어 등 모두 원문 그대로 유지.

---

# 3. Helm Chart 제작 및 Multi-Cluster 배포

### ▶ 실습 체크리스트

1. Helm Chart 디렉터리 생성
2. templates/configmap.yaml, deployment.yaml, service.yaml 작성
3. values-dev/prd.yaml 작성
4. Git push
5. Argo CD Application 생성(dev/prd/mgmt)
6. 각 클러스터 배포 결과 확인

## 실습 개요

이 섹션은 **환경별 설정 분리(Dev/Prd) + GitOps 배포 자동화**를 학습하는 매우 중요한 부분입니다.

Helm Chart를 한 번 정의해두면 코드 구조는 동일하게 유지하면서, values 파일을 환경별로 분리(`values-dev.yaml`, `values-prd.yaml`)하여 dev/prd 클러스터에 서로 다른 설정으로 배포할 수 있습니다.

예:

* Dev → replica 1, 포트 31000, 메시지 "Hello Dev"
* Prd → replica 2, 포트 32000, 메시지 "Hello Prd"

Argo CD Application에서 valueFiles만 다르게 지정하여 **하나의 Chart로 다중 환경을 자동 관리하는 GitOps 패턴**을 익히는 과정입니다.

* values-dev / values-prd 로 환경별 설정 분리
* Argo CD Application 3개 생성하여 mgmt/dev/prd 각각 배포

## 실습 — nginx-chart 생성

```
mkdir nginx-chart
cd nginx-chart
...
cat > templates/configmap.yaml <<EOF
...
EOF
```

Git push, helm template, argocd app 생성까지 전체 명령어 포함.

---

# 4. Argo CD ApplicationSet

### ▶ 실습 체크리스트

1. List Generator 기반 ApplicationSet 작성
2. dev/prd 자동 Application 생성 확인
3. Clusters Generator 기반 ApplicationSet 작성
4. label selector로 특정 클러스터만 배포
5. Application 자동 생성/삭제 동작 확인

## 4.1 실습 개요

ApplicationSet은 App-of-Apps 구조의 정적 한계를 극복하기 위한 **동적 Application 생성 컨트롤러**입니다.

핵심 특징:

* 여러 클러스터(dev/prd/stg/prod 등)에 자동으로 앱 배포
* 클러스터 추가 시 YAML 수정 없이 자동 반영
* Git 리포지터리 구조 변화도 유연하게 처리
* Multi-team/Multi-tenant 환경에서 스케일링에 최적화

실습에서는 List Generator와 Cluster Generator를 활용해 **클러스터 숫자에 따라 Application을 자동 생성하는 방식**을 체험합니다.

## 4.1 개념 요약

ApplicationSet은 App-of-Apps의 단점을 해결하는 자동 Application 생성 프레임워크.

* 제너레이터(List, Clusters, Git, SCM, Matrix 등)
* 동적 클러스터 자동 탐색
* Multi-tenant 구조 지원

## 4.2 실습 — ApplicationSet List Generator

전체 명령어 그대로:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
...
```

Sync → app 자동 생성 → 상태 확인까지 원문 그대로 유지합니다.

---

# 5. Keycloak + Argo CD + Jenkins (OIDC SSO)

### ▶ 실습 체크리스트

1. Keycloak 설치 및 서비스 노출
2. Keycloak Realm 생성
3. Argo CD OIDC Client 생성
4. Jenkins OIDC Client 생성
5. CoreDNS hosts 설정 적용
6. 브라우저에서 Argo CD → Jenkins SSO 흐름 확인 (OIDC SSO)

## 실습 개요

이 섹션은 엔터프라이즈 인증 환경의 핵심인 **통합 SSO 아키텍처**를 구축합니다.

* Keycloak은 중앙 IdP(Identity Provider)로 사용
* Argo CD와 Jenkins는 OIDC Client 역할
* 하나의 Keycloak Realm에서 통합 인증 제공 → 완전한 SSO 가능

즉,

1. 사용자가 Argo CD에 접속 → Keycloak 로그인 페이지로 이동
2. 로그인 성공 → 토큰 발급 → Argo CD 세션 생성
3. Jenkins에 접속하면 Keycloak 세션을 재사용해 자동 로그인

또한 CoreDNS hosts 플러그인을 사용하여 클러스터 내부에서도 Keycloak 도메인을 올바르게 해석하도록 구성해 OIDC Redirect/Callback 오류를 방지합니다.

## 실습 개요

* Keycloak = IdP
* Argo CD와 Jenkins는 모두 OIDC 지원
* 하나의 Keycloak Realm에서 통합 인증 제공 → 완전한 SSO 구성
* CoreDNS hosts 기능을 활용하여 cluster 내부에서도 도메인 기반 OIDC 동작 보장

## 실습 — Keycloak 설치 (쉘 그대로)

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
...
EOF
```

curl 테스트, ingress 설정, 도메인 hosts 등록 등 전체 명령어 그대로 보존.

### Argo CD 클라이언트 생성

```
kubectl -n argocd patch secret argocd-secret ...
```

### Jenkins OIDC 설정

설정 절차 + curl 테스트 + hosts 설정 모두 포함.

---

# 6. OpenLDAP + Keycloak 통합

### ▶ 실습 체크리스트

1. OpenLDAP 배포
2. people/groups OU 생성
3. 사용자·그룹 추가(alice, bob, devs, admins)
4. Keycloak User Federation(LDAP) 연결
5. 그룹 Mapper 생성
6. Token Claim(groups) 확인

## 실습 개요

OpenLDAP은 기업의 조직도/사용자/그룹 정보를 저장하는 디렉터리 서비스이며, Keycloak은 이를 User Federation으로 연동하여 LDAP 계정 기반 인증을 제공합니다.

LDAP 사용자·그룹을 Keycloak으로 가져오고, 이 정보를 Token Claim(groups)으로 전달하여 Argo CD와 Jenkins 등 서비스 레벨의 RBAC에 활용할 수 있습니다.

즉,

* LDAP → Keycloak 그룹 매핑
* Keycloak → OIDC Token(groups)
* Argo CD → RBAC(policy.csv)

으로 이어지는 완전한 엔터프라이즈 인증·권한 모델 흐름을 구축하게 됩니다.

## 실습 개요

LDAP = 조직도 형태의 사용자 DB.
Keycloak은 LDAP을 User Federation 으로 연동.
그룹/사용자 항목을 Token Claim(groups) 으로 전달.
Argo CD는 RBAC policy.csv에서 group 기반 권한 부여 가능.

## 실습 — OpenLDAP 배포 (쉘 그대로)

```
cat <<EOF | kubectl apply -f -
...
EOF
```

LDAP 엔트리 추가:

```
ldapadd -x -D "cn=admin,dc=example,dc=org" -w admin <<EOF
...
EOF
```

LDAP search, whoami, 그룹 생성 등 모든 명령어 그대로 포함.

## Keycloak LDAP Federation 설정

* Bind DN
* Users DN: ou=people,dc=example,dc=org
* Mapper: group-ldap-mapper

동기화 후 OIDC 토큰 Claim에 `groups` 추가.

---

# 7. Argo CD RBAC 적용

### ▶ 실습 체크리스트

1. Keycloak 그룹 devs에 사용자 추가
2. devs 그룹이 Token Claim에 포함되는지 확인
3. argocd-rbac-cm 수정 (g, devs, role:admin)
4. bob 계정으로 SSO 로그인
5. 관리자 권한 정상 동작 확인

## 실습 개요

이 섹션은 LDAP/Keycloak에서 넘어온 그룹 정보를 Argo CD의 RBAC 정책에 연결하여 **실제 권한 제어를 완성하는 단계**입니다.

Argo CD RBAC는 `policy.csv` 기반으로 동작하며 다음과 같이 정의됩니다:

* `p, subject, resource, action, allow/deny`
* `g, group, role` (그룹과 역할 연결)

예:

```
g, devs, role:admin
```

→ devs 그룹 사용자(bob)는 Argo CD admin 권한 획득

이 과정을 통해 기업 조직 LDAP → Keycloak → Argo CD 까지 이어지는 권한 체계를 직접 구현해봅니다.

```
KUBE_EDITOR="nano" kubectl edit cm argocd-rbac-cm -n argocd
...
policy.csv: |
  g, devs, role:admin
```

bob(devs 그룹) → Argo CD admin 권한 확인.

---

# 8. 클러스터 전체 삭제

### ▶ 실습 체크리스트

1. mgmt 삭제
2. dev 삭제
3. prd 삭제
4. Docker 네트워크/컨테이너 잔여물 확인
5. 재실습 시 전체 스크립트 재실행

## 실습 개요

실습을 마친 후 환경을 복원하려면 kind 클러스터를 완전히 삭제해야 합니다.
kind는 Docker 기반이기 때문에 삭제 후 다시 동일 스크립트를 실행하면 항상 동일한 구조가 재현됩니다.

이는 곧 **Infrastructure as Code(GitOps)** 사고방식을 자연스럽게 익히는 훈련입니다.

```
kind delete cluster --name mgmt ; kind delete cluster --name dev ; kind delete cluster --name prd
```

---

# 문서 설명

이 문서는:

* 개념/원리 설명은 풍부하게 재구성
* 실습 부분은 사용자의 **쉘 명령·yaml·출력 로그를 원본 그대로 보존**
* 다음에 다시 실습해도 완벽히 재현 가능하도록 구성된 통합본입니다.
