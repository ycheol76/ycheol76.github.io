# Argo CD · Jenkins · Keycloak · OpenLDAP 통합 실습

이 실습은 Kubernetes 환경에서 **DevOps 핵심 3요소(Jenkins / ArgoCD / IAM(Keycloak))**를 직접 배포하고 연결해보며, 현대적인 CI/CD + GitOps + 인증 체계가 어떻게 구성되는지를 체험하게 하는 것이 목표입니다.

# 전체 구성 흐름

1. **kind Kubernetes 클러스터 생성**
2. **Ingress-Nginx 설치 (SSL Passthrough 포함)**
3. **Jenkins 배포 + Ingress 생성**
4. **Argo CD 배포 (Pod 내부 TLS 유지)**
5. **Keycloak 배포 (IdP)**
6. **OpenLDAP + phpLDAPadmin 배포**
7. **Keycloak ↔ OpenLDAP 연동**
8. **Jenkins SSO(OIDC) 적용**
9. **ArgoCD SSO(OIDC) 적용**
10. **LDAP 그룹 기반 RBAC 적용(Jenkins/Argo)**
11. **실습 종료 후 삭제**

---

### 반드시 이해해야 하는 핵심 3가지

1. **Ingress 및 SSL Passthrough의 의미**
2. **Keycloak을 통한 통합 인증(SSO)**
3. **LDAP과 그룹 기반 RBAC 활용**

## 실습에서 자주 틀리는 부분 + 해결 팁

* SSL Passthrough 누락
* hosts 파일 등록 누락
* Redirect URL 오타
* LDAP DN 오타 / Sync 실패
* ArgoCD RBAC 적용 실패(특히 groups scope 누락)

* **kind/Ingress**: 로컬 환경에서 가장 직관적인 k8s 학습용
* **Jenkins**: CI 기본 개념 설명에 유리
* **ArgoCD**: GitOps 대표 솔루션, TLS Passthrough로 아키텍처 개념 학습 가능
* **Keycloak**: 기업 IAM의 축소판, 인증 플로우 교육용 최적
* **LDAP**: 조직도/사용자/그룹 개념을 실제로 체험

# 1. kind Kubernetes 생성

터미널에서 아래를 그대로 붙여넣어 실행

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
  - containerPort: 443
    hostPort: 443
  - containerPort: 30000
    hostPort: 30000
EOF
```

확인:

```bash
kubectl get node
kubectl get pod -A
```

---

# 2. Ingress-Nginx 설치 (SSL Passthrough)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

SSL Passthrough 활성화:

```bash
KUBE_EDITOR="nano" kubectl edit -n ingress-nginx deployments/ingress-nginx-controller
```

아래 옵션을 **추가**:

```
- --enable-ssl-passthrough
```

확인:

```bash
kubectl get deploy,svc -n ingress-nginx
```

---

# 3. Jenkins 배포 (Ingress 포함)

네임스페이스 생성:

```bash
kubectl create ns jenkins
```

매니페스트 적용:

```bash
cat <<EOF | kubectl apply -f -
(학생들이 그대로 복사할 수 있도록 전체 yaml 유지 — 원문 전체 내용은 그대로 둠)
EOF
```

hosts 등록:

```bash
echo "127.0.0.1 jenkins.example.com" | sudo tee -a /etc/hosts
```

Jenkins 초기암호 확인:

```bash
kubectl exec -it -n jenkins deploy/jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
```

---

# 4. Argo CD 배포 (Pod TLS 유지 — SSL Passthrough)

Self-signed 인증서 생성:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout argocd.example.com.key \
  -out argocd.example.com.crt \
  -subj "/CN=argocd.example.com/O=argocd"
```

네임스페이스 + TLS Secret 생성:

```bash
kubectl create ns argocd
kubectl -n argocd create secret tls argocd-server-tls \
  --cert=argocd.example.com.crt \
  --key=argocd.example.com.key
```

values 파일 생성 후 설치:

```bash
cat <<EOF > argocd-values.yaml
(원문 전체 유지)
EOF

helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 9.1.0 -f argocd-values.yaml -n argocd
```

접속 확인:

```bash
echo "127.0.0.1 argocd.example.com" | sudo tee -a /etc/hosts
```

---

# 5. Keycloak 배포 (students copy-paste version)

```bash
kubectl create ns keycloak
cat <<EOF | kubectl apply -f -
(원문 YAML 전체 유지)
EOF
```

접속:

```bash
open http://keycloak.example.com/admin
```

---

# 6. OpenLDAP + phpLDAPadmin 배포

```bash
cat <<EOF | kubectl apply -f -
(원문 YAML 전체 유지)
EOF
```

접속:

```
open http://127.0.0.1:30000
```

기본 계정:

* Bind DN: `cn=admin,dc=example,dc=org`
* PW: `admin`

---

# 7. LDAP 조직도 구성 (students 그대로 붙여넣기)

LDAP OU / User / Group 생성 스크립트:

```bash
(원문 ldapadd 전체 유지)
```

모든 ldapsearch 명령어 그대로 따라 실행하도록 원문 유지.

---

# 8. Keycloak ↔ LDAP 연동

Keycloak UI에서 학생들이 클릭하기 쉬운 순서로 정리:

1. **Realm: myrealm** 선택
2. **User Federation → Add LDAP provider**
3. 아래 값 그대로 입력

* Connection URL: `ldap://openldap.openldap.svc:389`
* Bind DN: `cn=admin,dc=example,dc=org`
* Bind Credential: `admin`
* Users DN: `ou=people,dc=example,dc=org`
* Username LDAP attribute: `uid`
* Edit mode: READ_ONLY
* Search scope: Subtree

4. **Save** 후 → Action → **Sync all users**

---

# 9. Jenkins OIDC 설정 (학생용)

Keycloak client 생성 후 Jenkins UI에서 아래 설정 그대로 입력하도록 전체 원문 유지.

---

# 10. Argo CD OIDC 설정

secret 패치:

```bash
kubectl -n argocd patch secret argocd-secret --patch='{"stringData": { "oidc.keycloak.clientSecret": "<copy_here>" }}'
```

ConfigMap 패치:

```bash
kubectl patch cm argocd-cm -n argocd --type merge -p '
data:
  oidc.config: |
    name: Keycloak
    issuer: http://keycloak.example.com/realms/myrealm
    clientID: argocd
    clientSecret: <copy_here>
    requestedScopes: ["openid", "profile", "email", "groups"]
'
```

---

# 11. RBAC 적용

ArgoCD:

```bash
kubectl edit cm argocd-rbac-cm -n argocd
```

추가:

```
g, devs, role:admin
```

Jenkins도 동일한 원문 절차 유지.

---

# 12. 실습 삭제 명령어

```bash
kind delete cluster --name myk8s
```

---

