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

## ▶ 실습 체크리스트

* mgmt 클러스터 생성(kind)
* Ingress-NGINX 설치
* SSL Passthrough 활성화
* Argo CD 설치 및 초기 패스워드 확인
* 브라우저 접속 확인

## 1.1 실습 개요

이 섹션은 GitOps 컨트롤 플레인 구조를 이해하는 핵심 단계입니다. kind 클러스터를 3개(mgmt/dev/prd)로 구성하여 실제 엔터프라이즈 환경의 멀티 클러스터 운영 방식을 재현합니다.

* mgmt 클러스터: Argo CD, Ingress, SSO(Keycloak), CI(Jenkins)를 모두 탑재한 중앙 제어 클러스터
* dev/prd 클러스터: 애플리케이션 배포 대상. Argo CD가 GitOps 방식으로 관리
* 모든 클러스터는 Docker bridge 네트워크(kind)를 통해 통신하며, mgmt에서 dev/prd로 API 요청을 전달

즉, 다음과 같은 구조를 직접 구축하는 실습입니다:

Git → Argo CD(mgmt) → 여러 Kubernetes Cluster(dev/prd)

## 1.2 실습 — kind mgmt 배포

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

### TLS Secret 등록

```bash
kubectl -n argocd create secret tls argocd-server-tls \
  --cert=argocd.example.com.crt \
  --key=argocd.example.com.key
```

### Argo CD 설치

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set server.ingress.enabled=true \
  --set server.ingress.hostname=argocd.example.com \
  --set server.ingress.tls=true \
  --set server.ingress.extraTls[0].hosts[0]=argocd.example.com \
  --set server.ingress.extraTls[0].secretName=argocd-server-tls
```

### 초기 비밀번호 확인

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

# 2. dev / prd 클러스터 설치 및 kubeconfig 조정

## ▶ 실습 체크리스트

* dev/prd 클러스터 생성
* Docker 네트워크(kind bridge) 확인
* kubeconfig server IP(127.0.0.1 → 172.20.x.x) 변경
* Argo CD에 dev/prd 등록
* 정상 연결 확인

## 2.2 dev/prd 클러스터 생성 명령어

```bash
kind create cluster --name dev --image kindest/node:v1.32.8
kind create cluster --name prd --image kindest/node:v1.32.8
```

## 2.3 Docker 네트워크 확인

```bash
docker network inspect kind
```

## 2.4 dev/prd kubeconfig 수정

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dev-control-plane
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' prd-control-plane
```

`~/.kube/config` 파일에서 server 주소를 다음과 같이 변경:

```yaml
server: https://<위에서 구한 IP>:6443
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

## ▶ 실습 체크리스트

* Helm Chart 디렉터리 생성
* templates/configmap.yaml, deployment.yaml, service.yaml 작성
* values-dev/prd.yaml 분리 작성
* Git push
* Argo CD Application 생성(mgmt/dev/prd)
* 정상 배포 확인

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

## 3.3 Helm lint

```bash
helm lint ./chart
```

## 3.4 Application manifest 예시 (dev)

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
    server: https://<dev-cluster-ip>:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 4. Argo CD ApplicationSet

## ▶ 실습 체크리스트

* List Generator 기반 ApplicationSet 작성
* dev/prd 자동 Application 생성
* Cluster Generator로 label selector 구성
* 자동 생성/삭제 확인

## 4.2 실습 — List Generator 예시

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
            url: https://<dev-cluster-ip>:6443
          - cluster: prd-k8s
            url: https://<prd-cluster-ip>:6443
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

## ▶ 실습 체크리스트

* Keycloak 설치
* Realm 생성
* Argo CD / Jenkins OIDC Client 생성
* CoreDNS hosts 설정
* 브라우저에서 SSO 동작 확인

## 5.2 Keycloak 설치

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

## ▶ 실습 체크리스트

* OpenLDAP 배포
* people / groups OU 생성
* 사용자/그룹 추가
* Keycloak User Federation 연결
* 그룹 Mapper 추가
* Token Claim(groups) 확인

## 6.2 OpenLDAP 배포

```bash
docker run -d --name openldap \
  -p 389:389 \
  -e LDAP_ORGANISATION="demo" \
  -e LDAP_DOMAIN="demo.local" \
  -e LDAP_ADMIN_PASSWORD=admin \
  osixia/openldap
```

## 6.3 LDAP 구조 예시

* Base DN: `dc=demo,dc=local`
* Users DN: `ou=people,dc=demo,dc=local`
* Groups DN: `ou=groups,dc=demo,dc=local`

Keycloak에서 연결 후 테스트 사용자 로그인 및 그룹 claim 확인.

---

# 7. Argo CD RBAC 적용

## ▶ 실습 체크리스트

* Keycloak에서 group claim 포함 확인
* Argo CD에서 RBAC 정책 설정 (ConfigMap)
* `readonly`, `admin`, `project-admin` 등 역할 분리
* 사용자별 로그인 후 권한 차이 확인

## 실습 명령어 및 설정

```bash
# Argo CD RBAC ConfigMap 편집
kubectl -n argocd edit configmap argocd-rbac-cm
```

```yaml
# 예시
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

```bash
# 설정 적용 후 Argo CD 서버 재시작
kubectl -n argocd rollout restart deployment argocd-server
```

```bash
# Token 내 group claim 확인 (예: dev-admins)
jwt.io 에서 확인하거나 `kubectl get secret` 후 디코딩
```

---

# 8. 클러스터 삭제

## ▶ 실습 체크리스트

* dev/prd 클러스터 삭제
* mgmt 클러스터 삭제
* Docker 네트워크 정리

## 실습 명령어

```bash
kind delete cluster --name dev
kind delete cluster --name prd
kind delete cluster --name mgmt
```

```bash
docker ps -a
docker stop keycloak openldap
docker rm keycloak openldap
```

```bash
docker network prune
```
---

# 9. ApplicationSet Cluster Generator 고급 설정

## ▶ 실습 체크리스트

* kind 클러스터에 label 추가 (`argocd.argoproj.io/secret-type=cluster`)
* Cluster Generator에서 label selector 적용
* 클러스터를 동적으로 감지하고 자동 Application 생성

## 클러스터 라벨 설정 예시

```bash
kubectl -n argocd get secret -l 'argocd.argoproj.io/secret-type=cluster'

# 예: kind-dev 클러스터에 label 추가
kubectl -n argocd label secret <클러스터-시크릿명> env=dev --overwrite
kubectl -n argocd label secret <클러스터-시크릿명> env=prd --overwrite
```

## Cluster Generator ApplicationSet 예시

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
```

---

# 10. Argo CD Notifications 연동

## ▶ 실습 체크리스트

* Argo CD Notifications Controller 설치
* 슬랙 Webhook URL 구성
* Trigger & Template 설정
* App 상태 변화 시 알림 전송 확인

## 설치

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/stable/manifests/install.yaml
```

## 슬랙 Webhook Secret 등록

```bash
kubectl -n argocd create secret generic argocd-notifications-secret \
  --from-literal=slack-token=<your-webhook-url>
```

## configMap 설정 예시

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
labels:
  app.kubernetes.io/name: argocd-notifications-cm
  app.kubernetes.io/part-of: argocd
  notifications.argoproj.io/subscribe.on-created.slack: general
  notifications.argoproj.io/subscribe.on-deployed.slack: general
  notifications.argoproj.io/subscribe.on-health-degraded.slack: general
```

---

# 11. GitOps Pull vs Push 방식 비교

| 항목     | Pull 방식 (Argo CD) | Push 방식 (Jenkins 등) |
| ------ | ----------------- | ------------------- |
| 트리거    | Git 변경 감지         | CI 파이프라인 수동 트리거     |
| 보안     | Cluster 내부 → Git  | 외부 시스템 → Cluster    |
| 실습 난이도 | 낮음                | 높음 (권한 구성 필요)       |

## 실습: Jenkins에서 push 배포 예시

```groovy
pipeline {
  stages {
    stage('Deploy') {
      steps {
        sh 'kubectl apply -f k8s/deployment.yaml'
      }
    }
  }
}
```

---

# 12. Helm + Parameterization 자동화

## ▶ 실습 체크리스트

* `values.yaml` 템플릿화 (`values-dev.yaml`, `values-prd.yaml`)
* ApplicationSet에서 변수 사용 (`values-{{cluster}}.yaml`)
* Git 브랜치, 경로, 파일 분기 처리

## Helm Chart 내 values-dev.yaml 예시

```yaml
replicaCount: 1
image:
  repository: nginx
  tag: dev
```

## ApplicationSet 템플릿 예시

```yaml
helm:
  valueFiles:
    - values-{{name}}.yaml
```
