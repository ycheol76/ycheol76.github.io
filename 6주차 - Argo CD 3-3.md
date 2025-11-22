# 6주차 - Argo CD 3/3

# 목차

1. [Cluster Management — kind + Ingress + Argo CD](#1-cluster-management--kind--ingress--argo-cd)
2. [dev / prd 클러스터 설치 및 kubeconfig 조정](#2-dev--prd-클러스터-설치-및-kubeconfig-조정)
3. [Helm Chart 제작 및 Multi-Cluster 배포](#3-helm-chart-제작-및-multi-cluster-배포)
4. [Argo CD ApplicationSet](#4-argo-cd-applicationset)
5. [Keycloak + Argo CD + Jenkins (OIDC SSO)](#5-keycloak--argo-cd--jenkins-oidc-sso)
6. [OpenLDAP + Keycloak 통합](#6-openldap--keycloak-통합)
7. [Argo CD RBAC 적용](#7-argo-cd-rbac-적용)
8. [클러스터 삭제](#8-클러스터-삭제)
9. [ApplicationSet Cluster Generator 고급 설정](#9-applicationset-cluster-generator-고급-설정)
10. [Argo CD Notifications 연동](#10-argo-cd-notifications-연동)
11. [GitOps Pull vs Push 방식 비교](#11-gitops-pull-vs-push-방식-비교)
12. [Helm + Parameterization 자동화](#12-helm--parameterization-자동화)

---

# 1. Cluster Management — kind + Ingress + Argo CD

## 학습 목적(Why)

단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있습니다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요합니다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식입니다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적입니다.
GitOps 전체 구조에서 **중앙 제어(Control-plane) 클러스터를 별도로 두는 이유**를 이해하기 위함입니다.

* Argo CD, Keycloak, Jenkins 같은 ‘관리용 컴포넌트’는 개발/운영 서비스 클러스터와 분리해야 함
* 보안상 위험을 분리하고, 장애 영향범위를 줄이기 위함

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소입니다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커집니다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 합니다.

* SK hynix / 삼성 / 네이버 클라우드 모두 `Mgmt Cluster → Dev/Prod Cluster` 구조를 사용
* 운영/배포/보안 체계를 하나의 중앙 제어에서 관리해야 대규모 인프라 운영 가능

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있습니다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있습니다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생합니다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있습니다.

* ArgoCD가 dev/prd 안에 설치되면 해당 클러스터 장애 시 GitOps 전체가 정지
* 인증/배포/모니터링 기능이 함께 다운 → 전체 서비스 운영 중단

## 구조적 원리 (Architecture)

```
Git → ArgoCD (Mgmt) → Dev Cluster
                      → Prd Cluster
```

ArgoCD는 Kubernetes API Server와 gRPC로 통신하며 각 클러스터를 ‘원격 제어’함.
여기서 mgmt는 *제어 신호만 보내고* dev/prd는 *실제 워크로드만 수행*.

## 실무 모범 운영 방식

* Mgmt는 고가용성: 3 노드 구성 또는 EKS/GKE 같은 Managed control-plane 사용
* Dev/Prd는 독립 배포 → 서로 장애 영향 없음
* ArgoCD는 절대 workload 클러스터에 설치하지 않음

## 이걸 왜 배우는가?

GitOps 환경에서 **중앙 제어 클러스터(mgmt)** 와 **실제 워크로드 클러스터(dev/prd)** 를 분리하는 것은 엔터프라이즈 구조에서 가장 중요한 핵심 개념입니다.

### 반드시 이해해야 하는 이유

* **보안 분리:** 운영(관리) 환경과 서비스(업무) 환경을 분리 → 사고 시 영향 최소화
* **확장성:** 수십~수백 개의 클러스터를 하나의 Control-plane(Argo CD)으로 관리 가능
* **자동화의 기반:** ApplicationSet, RBAC, SSO 등 고급 기능의 기본 전제
* **실제 기업 운영 방식과 동일:** AWS, Naver, SK hynix, 삼성 클라우드 등 모두 동일 구조 사용

그래서 이번 섹션은 단순히 클러스터를 만드는 실습이 아니라,
**엔터프라이즈 GitOps 아키텍처를 몸으로 익히는 과정**임.
— kind + Ingress + Argo CD

## ▶ 실습 체크리스트

* mgmt 클러스터 생성(kind)
* Ingress-NGINX 설치
* SSL Passthrough 활성화
* Argo CD 설치 및 초기 패스워드 확인
* 브라우저 접속 확인

## 1.1 실습 개요

kind 클러스터를 mgmt/dev/prd 구조로 구성하여 실제 엔터프라이즈 GitOps 시나리오를 구현합니다:

* **mgmt:** GitOps Control-plane (Argo CD + Ingress + Keycloak)
* **dev/prd:** 애플리케이션이 실제로 배포되는 대상 클러스터
* **Argo CD는 mgmt에서 실행되며 dev/prd 클러스터의 리소스를 원격 제어**합니다.

Git → Argo CD(mgmt) → dev/prd Kubernetes

## 1.2 kind mgmt 클러스터 생성

```bash
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

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### SSL Passthrough 활성화

```bash
kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml \
| sed '/- --publish-status-address=localhost/a\\n        - --enable-ssl-passthrough' \
| kubectl apply -f -
```

## 1.4 Argo CD 설치

### TLS 인증서 생성

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout argocd.example.com.key \
  -out argocd.example.com.crt \
  -subj "/CN=argocd.example.com/O=argocd"
```

### Secret 등록

```bash
kubectl -n argocd create secret tls argocd-server-tls \
  --cert=argocd.example.com.crt \
  --key=argocd.example.com.key
```

### Helm 설치

```bash
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set server.ingress.enabled=true \
  --set server.ingress.hostname=argocd.example.com \
  --set server.ingress.tls=true \
  --set server.ingress.extraTls[0].hosts[0]=argocd.example.com \
  --set server.ingress.extraTls[0].secretName=argocd-server-tls
```

### 초기 비밀번호

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

# 2. dev / prd 클러스터 설치 및 kubeconfig 조정

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있습니다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해집니다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식입니다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적입니다.
운영 환경에서는 **여러 개의 Kubernetes 클러스터를 하나의 Argo CD로 제어**해야 합니다.
이를 위해 dev/prd 클러스터를 생성하고 Argo CD가 접근할 수 있도록 kubeconfig를 조정하는 과정을 학습합니다.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 실서비스(PRD)와 개발/테스트(DEV)를 철저하게 분리하는 것이 보안 표준
* 장애 발생 시 PRD로 영향 확산을 방지하는 구조 운영 필요
* ArgoCD는 여러 클러스터를 동시에 관리할 수 있어야 함

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* kubeconfig를 127.0.0.1 로 남겨두면 ArgoCD가 dev/prd에 접근 불가
* 클러스터 인증서/주소 변경을 잘못 처리하면 전체 배포 실패

## 구조적 원리

kind 클러스터는 Docker 내부에서 동작하므로 다음 흐름을 이해해야 함:

```
ArgoCD Pod → Docker Bridge Network → dev-control-plane → Kube-API
```

즉, `docker inspect` 로 얻은 내부 IP를 kubeconfig에 적용해야 ArgoCD가 접근 가능.

## 실무 모범 운영 방식

* 각 클러스터의 kubeconfig를 GitOps 저장소에 저장하지 않음(보안 위험)
* ArgoCD CLI 또는 UI를 통해 cluster secret 등록
* Dev/Prd는 label로 환경 구분 (env=dev, env=prd)

## ▶ 실습 체크리스트

* dev/prd 클러스터 생성
* Docker 네트워크(kind bridge) IP 확인
* kubeconfig server 주소 수정
* Argo CD cluster add

## 2.2 클러스터 생성

```bash
kind create cluster --name dev --image kindest/node:v1.32.8
kind create cluster --name prd --image kindest/node:v1.32.8
```

## 2.3 Docker 네트워크 확인

```bash
docker network inspect kind
```

## 2.4 kubeconfig 수정

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dev-control-plane
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' prd-control-plane
```

kubeconfig:

```yaml
server: https://<IP>:6443
```

## 2.5 Argo CD에 dev/prd 등록

```bash
argocd login argocd.example.com
argocd cluster add kind-dev --name dev-k8s
argocd cluster add kind-prd --name prd-k8s
argocd cluster list
```

---

# 3. Helm Chart 제작 및 Multi-Cluster 배포

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
Helm은 Kubernetes 애플리케이션 배포의 표준 템플릿 시스템.
Multi-cluster GitOps에서는 환경별 다른 값을 자동으로 주입해야 하므로 Helm은 필수.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* Dev/Prd 환경마다 replica, 이미지 태그, 도메인, 리소스 등이 다름
* Helm의 values-dev.yaml, values-prd.yaml 분리가 실무 표준

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* values.yaml 분리를 하지 않으면 PRD에 DEV 설정이 배포되는 대형 사고 발생
* 예: dev 이미지를 실수로 PRD에 배포 → 서비스 전체 장애

## 구조적 원리

Helm은 다음과 같이 템플릿(Rendering)을 수행:

```
Chart + values.yaml → rendered manifest → Kubernetes apply
```

ArgoCD는 Helm을 내장하여 자동으로 템플릿을 렌더링한다.

## 실무 모범 운영 방식

* values-dev.yaml / values-prd.yaml / values-stg.yaml 등 환경별 파일 구분
* ArgoCD syncPolicy는 dev는 자동, prd는 manual 운영

## ▶ 실습 체크리스트

* Chart 생성
* values-dev/prd.yaml 분리
* Git push 후 Application 생성

## 3.2 Chart 구조

```bash
chart/
 ├── Chart.yaml
 ├── values.yaml
 ├── values-dev.yaml
 ├── values-prd.yaml
 └── templates/
      ├── configmap.yaml
      ├── deployment.yaml
      └── service.yaml
```

## 3.4 Application(dev)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-app
spec:
  project: default
  source:
    repoURL: https://github.com/example-org/chart-repo
    targetRevision: HEAD
    path: chart
    helm:
      valueFiles:
      - values-dev.yaml
  destination:
    server: https://<dev-ip>:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 4. Argo CD ApplicationSet

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
수십~수백 개의 클러스터 또는 서비스에 대해 **자동 Application 생성/삭제** 를 하기 위해 ApplicationSet을 사용.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 클러스터 50개를 매번 수동 Application 등록하는 것은 불가능
* 신규 클러스터가 추가되면 자동으로 GitOps 배포가 구성되어야 함

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* List Generator만 사용하면 확장성 부족
* Cluster Generator를 사용하지 않으면 다중 클러스터 자동화 불가능

## 구조적 원리

ApplicationSet → Generator가 클러스터 목록을 불러옴 → Template으로 Application 생성

예:

```
Cluster Secret (label=env=dev)
        ↓ 자동 인식
ApplicationSet → dev-app 생성
```

## 실무 모범 운영 방식

* Cluster Generator + Label selector 사용
* ApplicationSet은 GitOps 배포의 표준 패턴으로 구성

## ▶ 실습 체크리스트

* List Generator
* Cluster Generator

## 4.2 List Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-demo
spec:
  generators:
    - list:
        elements:
          - cluster: dev-k8s
            url: https://<dev-ip>:6443
          - cluster: prd-k8s
            url: https://<prd-ip>:6443
  template:
    metadata:
      name: '{{cluster}}-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/example-org/chart-repo
        targetRevision: HEAD
        path: chart
        helm:
          valueFiles:
            - values-{{cluster}}.yaml
      destination:
        server: '{{url}}'
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

# 5. Keycloak + Argo CD + Jenkins (OIDC SSO)

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
엔터프라이즈 보안에서 SSO는 선택이 아니라 필수.
ArgoCD·Jenkins·Harbor 등 모든 DevOps 도구는 Keycloak을 통한 OIDC 통합 인증을 사용.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 계정, 권한, 인증을 중앙 통합 관리해야 보안 사고 방지
* 퇴사/이동 시 시스템별 계정 삭제할 필요 없음

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* SSO 설정 오류 → Argo CD 로그인 불가 → GitOps 운영 중단
* Redirect URI 오타 → 인증 루프 발생

## 구조적 원리

OIDC 흐름:

```
User → ArgoCD → Keycloak 로그인 → Token 발급 → ArgoCD 세션 생성
```

## 실무 모범 운영 방식

* 모든 DevOps 시스템은 반드시 OIDC 통합
* 그룹 → 역할 → RBAC 매핑 구조 유지
  (OIDC SSO)

## ▶ 실습 체크리스트

* Keycloak 설치
* Realm / Client 생성
* Argo CD / Jenkins OIDC 로그인

## Keycloak 설치

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:23.0.6 \
  start-dev
```

---

# 6. OpenLDAP + Keycloak 통합

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
기업에서는 명확한 계정 체계를 위해 LDAP/AD 기반 사용자 DB를 사용.
Keycloak은 LDAP을 페더레이션하여 인증/권한을 통합.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 사내 조직/부서/직급에 기반한 권한 제어 가능
* 그룹 기반 RBAC 자동화

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* LDAP DN/DN 구조를 잘못 입력 → 인증 실패
* Group mapper 오류 → ArgoCD 권한 오작동

## 구조적 원리

LDAP 계정 → Keycloak User Federation → OIDC Token → ArgoCD RBAC

## 실무 모범 운영 방식

* Groups → Keycloak roles → ArgoCD RBAC
* Token에 groups claim 반드시 포함

## LDAP 배포

```bash
docker run -d --name openldap \
  -p 389:389 \
  -e LDAP_ORGANISATION="demo" \
  -e LDAP_DOMAIN="demo.local" \
  -e LDAP_ADMIN_PASSWORD=admin \
  osixia/openldap
```

LDAP 구조:

* `dc=demo,dc=local`
* `ou=people,dc=demo,dc=local`
* `ou=groups,dc=demo,dc=local`

---

# 7. Argo CD RBAC 적용

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
GitOps에서 가장 중요한 보안 개념은 **RBAC 기반 접근 제어**.
누가 어느 프로젝트를 보고/배포할 수 있는지 명확히 정의해야 함.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 다중 팀 환경에서 프로젝트 단위 권한 분리가 필수
* 잘못된 권한 설정은 대규모 장애를 초래

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* 모든 사용자를 admin으로 설정 → 최악의 보안 사고
* 특정 팀이 다른 팀 리소스를 실수로 수정

## 구조적 원리

Keycloak groups → ArgoCD RBAC roles → 정책 적용

## 실무 모범 운영 방식

* 최소 권한 원칙(Principle of least privilege)
* admin / project-admin / readonly 3단계 구조

## RBAC ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    g, dev-admins, role:admin
    g, dev-viewers, role:readonly
```

서버 재시작:

```bash
kubectl -n argocd rollout restart deployment argocd-server
```

---

# 8. 클러스터 삭제

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
GitOps 실습 환경을 종료할 때 안전하게 리소스를 정리하는 방법을 배움.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 클러스터/리소스가 남아 있으면 비용 청구 또는 보안 위험 발생

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* 잘못된 클러스터 삭제 → 운영 클러스터 삭제 위험

## 구조적 원리

kind cluster는 Docker container로 구성되어 있어 삭제가 매우 단순함.

## 실무 모범 운영 방식

* PRD는 절대 자동 삭제 안 함
* 실습 환경만 삭제

```bash
kind delete cluster --name dev
kind delete cluster --name prd
kind delete cluster --name mgmt
```

---

# 9. ApplicationSet Cluster Generator 고급 설정

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
Cluster Generator를 사용하여 **새로운 클러스터 자동 인식 + 자동 Application 배포** 구조를 학습.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 새로운 클러스터가 추가될 때 자동화가 반드시 필요

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* label 관리 실수 → 잘못된 클러스터에 배포

## 구조적 원리

Cluster Secret(label=env=dev) → ApplicationSet 자동 스캔 → dev-app 자동 생성

## 실무 모범 운영 방식

* cluster secret naming 규칙 통일
* label 기반 환경 구분

## Secret 라벨 추가

```bash
kubectl -n argocd label secret <name> env=dev --overwrite
```

## Cluster Generator 예시

````yaml
apiVersion: argoproj.io/v1

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: dev
  template:
    metadata:
      name: '{{name}}-demo-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/example-org/chart-repo
        targetRevision: HEAD
        path: chart
        helm:
          valueFiles:
            - values-dev.yaml
      destination:
        server: '{{server}}'
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
````

---

# 10. Argo CD Notifications 연동

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
GitOps 운영에서 배포 성공/실패/드리프트를 Slack·Teams로 알림받는 체계를 구축.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 운영자가 실시간으로 배포 상태를 파악해야 함
* PRD 장애 대응 시간 감소

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* 알림 미구성 → 배포 실패를 뒤늦게 발견

## 구조적 원리

ArgoCD Event → Notifications controller → Slack Webhook → 메시지 발송

## 실무 모범 운영 방식

* on-deployed / on-sync-failed / on-health-degraded 알림 사용

## ▶ 실습 체크리스트

* Notifications Controller 설치
* Slack Webhook 설정
* Trigger & Template 구성
* 배포/오류 이벤트 알림 확인

## 설치

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/stable/manifests/install.yaml
```

## Slack Secret 등록

```bash
kubectl -n argocd create secret generic argocd-notifications-secret \
  --from-literal=slack-token=<your-webhook-url>
```

## Notifications CM 설정

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
labels:
  notifications.argoproj.io/subscribe.on-created.slack: general
  notifications.argoproj.io/subscribe.on-deployed.slack: general
  notifications.argoproj.io/subscribe.on-health-degraded.slack: general
```

---

# 11. GitOps Pull vs Push 방식 비교

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
두 방식의 차이를 이해해야 GitOps를 올바르게 설계 가능.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* 엔터프라이즈는 반드시 Pull 방식을 선호(GitOps 표준)

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* Push 방식 사용 시 잘못된 권한 설정으로 PRD 직접 변경 위험

## 원리 비교

### Pull 방식 (ArgoCD)

* 클러스터 내부에서 Git을 Pull → 보안적

### Push 방식 (Jenkins 등)

* 외부에서 kubectl apply → 위험도 높음

## 실무 모범 운영 방식

* Dev는 Push 허용, Prd는 Pull만 사용

| 항목       | Pull 방식(Argo CD)  | Push 방식(Jenkins 등) |
| -------- | ----------------- | ------------------ |
| 트리거      | Git 변경 감지         | CI Job 실행          |
| 보안       | Cluster 내부에서 pull | 외부에서 kubectl apply |
| Drift 감지 | 자동 self-heal      | 없음                 |
| 운영 난이도   | 낮음                | 높음                 |

### Push 배포 예시

```groovy
pipeline {
  stages {
    stage('Deploy') {
      steps {
        sh 'kubectl apply -f manifests/'
      }
    }
  }
}
```

---

# 12. Helm + Parameterization 자동화

## 학습 목적(Why)

본 섹션의 학습 목적은 단순한 기능 실습이 아니라, 엔터프라이즈 GitOps 구조를 실제로 구현할 수 있는 근본 개념을 체득하는 데 있다. 조직 규모가 커질수록 인프라는 복잡해지고, 팀 간 자원 공유·권한 관리·서비스 분리·보안 통제가 중요해진다. 이러한 상황에서 GitOps 방식은 운영의 예측 가능성, 변경 이력 관리, 자동화 수준을 극대화하는 유일한 방식이다. 각 섹션에서 학습하는 내용은 단순 Kubernetes 자원 배포를 넘어, 기업 수준의 클러스터 운영 체계를 이해하는 데 필수적이다.
환경별 파라미터 자동 분리를 이해하고 ArgoCD와 결합하여 완전 자동화를 구현.

## 실무에서 필수인 이유

이 개념은 실제 조직에서 반드시 요구되는 운영 안정성, 보안 통제, 서비스 분리, 지속 배포 체계를 갖추기 위한 핵심 요소다. 소규모 환경에서는 단일 클러스터에 모든 것을 넣어도 큰 문제가 없지만, 엔터프라이즈 환경에서는 개발과 운영, 서비스와 관리, 인증 시스템과 배포 시스템 모두를 분리하지 않으면 장애 시 전체 서비스가 중단될 위험이 커진다. 또한 팀 단위 RBAC, 배포 정책, SSO 통합 등은 필수적인 운영 요구사항이며, 이를 지원하려면 본 섹션에서 다루는 구조와 원리를 반드시 이해해야 한다.

* Dev/Prd 설정이 다르기 때문에 values 분리는 필수

## 잘못 이해했을 때의 리스크

이 내용을 충분히 이해하지 못하면 단일 장애점(SPOF), 전체 서비스 중단, 권한 오남용, 보안 취약점 노출, 배포 충돌 등 다양한 문제가 발생할 수 있다. 예를 들어 Helm values 분리 개념을 오해하면 개발용 설정이 운영 클러스터에 적용되어 전체 서비스 장애가 발생하는 대형 사고가 일어날 수 있다. ApplicationSet 개념을 오해하면 신규 클러스터 생성 시 배포가 누락되거나 잘못된 환경에 배포되는 실수가 발생한다. 인증/권한 개념을 잘못 적용하면 전체 인프라 접근 권한이 노출되어 보안 사고로 이어질 수 있다.

* values-dev.yaml 실수로 prd에 적용 → 대규모 사고

## 구조적 원리

ApplicationSet name 기준으로 values 자동 바인딩

## 실무 모범 운영 방식

* values-환경명.yaml 파일 구조 통일 (dev, stg, prd)

## ▶ 실습 체크리스트

* values-dev/prd.yaml 분리
* ApplicationSet에서 변수 템플릿 사용
* GitOps 환경별 분기 자동화

## values-dev.yaml

```yaml
replicaCount: 1
image:
  repository: nginx
  tag: dev
```

## ApplicationSet 템플릿

```yaml
helm:
  valueFiles:
    - values-{{name}}.yaml
```

---

# 🎉 정리

이번 6주차에서는 **멀티 클러스터 GitOps 운영**, **ApplicationSet 자동화**, **SSO 인증**, **RBAC**, **Notifications**까지 엔터프라이즈 GitOps에서 꼭 필요한 고급 개념을 모두 실습했습니다.

원하시면 다음으로,

* ApplicationSet 고가용성 운영 가이드
* GitOps 모놀리식/마이크로서비스 패턴 비교
* Argo CD 고급 Sync 전략
  도 이어서 정리해 드릴 수 있어요!
