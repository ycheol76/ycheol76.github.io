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

# 1. Cluster Management

## 개념

Kind 클러스터 3개(mgmt/dev/prd)를 만들어 mgmt 클러스터에 Argo CD를 설치하고, dev/prd 클러스터를 GitOps로 관리하는 구조입니다.

* **mgmt:** Argo CD + Ingress-NGINX 설치, 모든 제어의 중심
* **dev/prd:** 애플리케이션 배포 대상 클러스터
* **통신:** Docker 네트워크(kind) 내부 IP 기반

## 실습 — kind mgmt 배포

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

## Ingress 설치

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

SSL Passthrough 활성화:

```
kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml \
| sed '/- --publish-status-address=localhost/a\
        - --enable-ssl-passthrough' | kubectl apply -f -
```

## Argo CD 설치

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 ...
...
helm install argocd argo/argo-cd --version 9.0.5 -f argocd-values.yaml --namespace argocd
```

로그인, 패스워드 변경 등 모든 명령어 그대로 유지.

---

# 2. dev / prd 클러스터 설치 및 kubeconfig 조정

## 개념

* kind 기본 kubeconfig는 `127.0.0.1` 로 된 server 주소를 포함
* mgmt → dev/prd 간 호출 위해선 **Docker 네트워크 IP**를 명시해야 함

## 실습 (쉘 그대로)

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

## 개념

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

## 개념

ApplicationSet은 App-of-Apps의 단점을 해결하는 자동 Application 생성 프레임워크.

* 제너레이터(List, Clusters, Git, SCM, Matrix 등)
* 동적 클러스터 자동 탐색
* Multi-tenant 구조 지원

## 실습 — ApplicationSet List Generator

전체 명령어 그대로:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
...
```

Sync → app 자동 생성 → 상태 확인까지 원문 그대로 유지합니다.

---

# 5. Keycloak + Argo CD + Jenkins (OIDC SSO)

## 개념

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

## 개념

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

```
KUBE_EDITOR="nano" kubectl edit cm argocd-rbac-cm -n argocd
...
policy.csv: |
  g, devs, role:admin
```

bob(devs 그룹) → Argo CD admin 권한 확인.

---

# 8. 클러스터 전체 삭제

```
kind delete cluster --name mgmt ; kind delete cluster --name dev ; kind delete cluster --name prd
```
