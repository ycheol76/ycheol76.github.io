# 5장 헬름(Helm)

이 문서는 **설명 + 실습 명령**을 함께 담은 “복붙용 가이드”입니다. 각 절은 **왜 하는지(개념)** → **어떻게 하는지(명령)** → **무엇을 확인해야 하는지(검증/트러블슈팅)** 순서로 구성했습니다.

## 목차

* [소개](#소개)
* [용어 간단 정리](#용어-간단-정리)
* [사전 체크리스트](#사전-체크리스트)
* [5.1 Creating a Helm Project](#51-creating-a-helm-project)
* [5.2 Reusing Statements Between Templates](#52-reusing-statements-between-templates)
* [5.3 Updating an Application (이미지/값 갱신 & 롤백)](#53-updating-an-application-이미지값-갱신--롤백)
* [5.4 Packaging and Distributing a Helm Chart](#54-packaging-and-distributing-a-helm-chart)
* [5.5 Deploying a Chart from a Repository](#55-deploying-a-chart-from-a-repository)
* [5.6 Deploying a Chart with a Dependency](#56-deploying-a-chart-with-a-dependency)
* [5.7 Triggering a Rolling Update Automatically](#57-triggering-a-rolling-update-automatically)
* [Bitnami 공개 카탈로그 삭제](#bitnami-공개-카탈로그-삭제)
* [OCI (OCI Registry for Helm)](#oci-oci-registry-for-helm)
* [Bitnami Helm Charts (OCI 예제)](#bitnami-helm-charts-oci-예제)
* [6장 Cloud Native CI/CD](#6장-cloud-native-cicd)

  * [6.1 Install Tekton](#61-install-tekton)
  * [6.2 Create a Hello World Task](#62-create-a-hello-world-task)
  * [6.3 Create a Task to Compile and Package an App from Git](#63-create-a-task-to-compile-and-package-an-app-from-git)
  * [6.4 Create a Task to Compile and Package an App from Private Git](#64-create-a-task-to-compile-and-package-an-app-from-private-git)
  * [6.5 Containerize an Application Using a Tekton Task and Buildah](#65-containerize-an-application-using-a-tekton-task-and-buildah)
* [자주 하는 실수 & 베스트 프랙티스](#자주-하는-실수--베스트-프랙티스)
* [정리](#정리)
* [Cleanup](#cleanup)

---

## 소개

**Helm**은 쿠버네티스 리소스(YAML)를 **Go 템플릿**으로 묶어 **차트(Chart)**라는 배포 단위로 관리하는 **패키지 관리자**입니다.

* 우리가 원하는 값만 `values.yaml`/`--set`으로 바꿔서 일관되게 설치·업그레이드·롤백할 수 있습니다.
* 차트는 버전이 있으므로 **재현 가능한 배포**가 가능합니다.
* Kustomize는 패치/오버레이 중심이고 템플릿 언어가 없지만, Helm은 템플릿 함수(Sprig 포함)와 **의존성, 릴리스 메타데이터, 롤백**까지 제공합니다.
* 단, **ConfigMap 변경**은 기본적으로 파드를 다시 굴리지 않습니다. Helm에서는 템플릿에 **체크섬 어노테이션**을 주입해 변경을 감지하도록 합니다(§5.7).

---

## 용어 간단 정리

* **Chart**: 애플리케이션 배포 단위(템플릿 + 기본값 + 메타데이터).
* **Release**: 특정 차트를 특정 값으로 설치한 인스턴스. 릴리스 이력은 Secret(`sh.helm.release.*`)에 저장.
* **`Chart.yaml`**: 차트 이름, 버전(`version`), 앱 버전(`appVersion`) 등 메타 정보.

  * `version`(차트 자체 변경 시 갱신) vs `appVersion`(애플리케이션 버전) — 의미가 다릅니다.
* **`values.yaml`**: 템플릿에 주입할 기본값. `--set`, `-f other.yaml`로 오버라이드.
* **templates/**: Go 템플릿으로 된 쿠버네티스 매니페스트.
* **`_helpers.tpl`**: 여러 템플릿에서 **재사용**할 블록 정의 파일.

---

## 사전 체크리스트

* kubectl이 현재 클러스터에 정상 연결되는지: `kubectl version --client && kubectl get ns`
* Helm 3.x 설치되어 있는지: `helm version`
* (로컬 실습) kind/Minikube 등 클러스터 준비.

---

## 5.1 Creating a Helm Project

**왜?** 템플릿화된 차트를 만들면 배포가 반복 가능하고, 환경마다 값만 바꿔 배포할 수 있습니다.

### 실습: 차트 생성 → 로컬 렌더링 → 설치/검증

1. **kind(k8s) 배포** — 실습용 클러스터

```bash
kind create cluster --name myk8s --image kindest/node:v1.32.8 --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
EOF
```

2. **차트 스캐폴드**

```bash
mkdir -p pacman/templates
cd pacman
```

**Chart.yaml** — 차트/앱 버전 구분 주석 포함

```yaml
apiVersion: v2
name: pacman
description: A Helm chart for Pacman
type: application
version: 0.1.0        # 차트 버전(템플릿 등 차트 정의 변경 시)
appVersion: "1.0.0"   # 애플리케이션 버전(이미지 태그 의미로 흔히 사용)
```

**templates/deployment.yaml** — 값(`.Values`)과 메타 정보(`.Chart`)를 템플릿으로 참조

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    {{- if .Chart.AppVersion }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          ports:
            - containerPort: {{ .Values.image.containerPort }}
              name: http
              protocol: TCP
```

**templates/service.yaml** — 셀렉터/포트는 values에서 참조

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
  name: {{ .Chart.Name }}
spec:
  ports:
    - name: http
      port: {{ .Values.image.containerPort }}
      targetPort: {{ .Values.image.containerPort }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
```

**values.yaml** — 기본값 정의(환경마다 오버라이드 권장)

```yaml
image:
  repository: quay.io/gitops-cookbook/pacman-kikd
  tag: "1.0.0"
  pullPolicy: Always
  containerPort: 8080

replicaCount: 1
securityContext: {}
# 예)
# securityContext:
#   capabilities:
#     drop: ["ALL"]
#   readOnlyRootFilesystem: true
#   runAsNonRoot: true
#   runAsUser: 1000
```

**디렉터리 트리**

```
pacman/
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml
```

3. **로컬 렌더링** — 클러스터에 적용 전 템플릿 결과 확인

```bash
helm template . | sed -n '1,120p'
helm template --set replicaCount=3 . | grep -n 'replicas:' -n -A1
```

4. **설치 & 상태 확인**

```bash
helm install pacman .
helm list
kubectl get deploy,po,svc,ep
kubectl get pod -o yaml | kubectl neat | yq
kubectl get pod -o json | grep -A1 securityContext
helm history pacman
kubectl get secret | grep sh.helm.release
```

> **포인트**: Helm은 릴리스 메타데이터(차트, 값, 이력)를 Secret으로 저장하기 때문에, **복구/롤백**이 쉽습니다.

---

## 5.2 Reusing Statements Between Templates

**왜?** 템플릿에 같은 코드를 여러 번 복붙하면 유지보수 지옥입니다. 공통 블록을 만들어 한 곳에서 바꾸면 끝!

### 실습: `_helpers.tpl`로 라벨/셀렉터 중복 제거

현재 중복되는 부분(발췌):

```yaml
# deployment.yaml
selector:
  matchLabels:
    app.kubernetes.io/name: {{ .Chart.Name }}
...
labels:
  app.kubernetes.io/name: {{ .Chart.Name }}

# service.yaml
selector:
  app.kubernetes.io/name: {{ .Chart.Name }}
```

1. **_helpers.tpl** 생성

```gotemplate
{{- define "pacman.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
{{- end -}}
```

2. **include + nindent**로 삽입

```gotemplate
# deployment.yaml
selector:
  matchLabels:
    {{- include "pacman.selectorLabels" . | nindent 6 }}
...
labels:
  {{- include "pacman.selectorLabels" . | nindent 8 }}

# service.yaml
selector:
  {{- include "pacman.selectorLabels" . | nindent 2 }}
```

3. **한 번에 확장** — 버전 라벨 추가 후 재렌더

```gotemplate
{{- define "pacman.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}
```

```bash
helm template . | grep -n 'version:' -n -A1
```

> **포인트**: 공통 블록은 "한 방에 교체"가 가능해 실수/누락을 줄여줍니다.

---

## 5.3 Updating an Application (이미지/값 갱신 & 롤백)

**왜?** 운영 중인 애플리케이션의 **이미지/설정**을 안전하게 업데이트하고, 문제가 생기면 **즉시 롤백**하려고.

### 실습: 업그레이드 → 이력 확인 → 롤백 → values 오버라이드

1. 초기 상태 설치(도움 블록 원복)

```bash
cat > templates/_helpers.tpl <<'EOF'
{{- define "pacman.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
{{- end -}}
EOF
helm install pacman .
helm history pacman
kubectl get deploy -o wide
```

2. **이미지 1.1.0으로 업그레이드**

```bash
cat > values.yaml <<'EOF'
image:
  repository: quay.io/gitops-cookbook/pacman-kikd
  tag: "1.1.0"
  pullPolicy: Always
  containerPort: 8080
replicaCount: 1
securityContext: {}
EOF

cat > Chart.yaml <<'EOF'
apiVersion: v2
name: pacman
description: A Helm chart for Pacman
type: application
version: 0.1.0
appVersion: "1.1.0"
EOF

helm upgrade pacman .
helm history pacman
kubectl get secret | grep sh.helm.release
kubectl get deploy,replicaset -o wide
```

> `appVersion`(앱 버전)과 `version`(차트 버전)은 **의미가 다릅니다**. 템플릿이 바뀌면 차트 버전, 애플리케이션이 바뀌면 앱 버전을 올립니다.

3. **롤백** — 문제가 생기면 빠르게 복구

```bash
helm history pacman
helm rollback pacman 1 && kubectl get pod -w
helm history pacman
kubectl get deploy,replicaset -o wide
```

4. **values 오버라이드 파일** — 팀/환경별 값 분리

```bash
cat > newvalues.yaml <<'EOF'
image:
  tag: "1.2.0"
EOF
helm template pacman -f newvalues.yaml . | grep 'image:' -n -A1
```

---

## 5.4 Packaging and Distributing a Helm Chart

**왜?** 우리가 만든 차트를 **다른 팀/환경**에서 재사용하려면 배포 가능한 아티팩트(.tgz)와 인덱스가 필요합니다.

### 실습: 패키징 → 인덱스 생성(Helm repo 방식)

```bash
helm package .
# 결과: ./pacman-0.1.0.tgz

helm repo index .
cat index.yaml | sed -n '1,120p'
```

**예시 레이아웃**

```
repo/
├── index.yaml
└── pacman-0.1.0.tgz
```

> **참고**: Helm 전용 repo 대신 **OCI 레지스트리**에 차트를 저장/설치할 수도 있습니다(§OCI).

---

## 5.5 Deploying a Chart from a Repository

**왜?** “남이 만든 차트(예: Bitnami)를 공식 저장소에서 바로 설치”하면 표준화/속도/보안 이점이 큽니다.

### 실습 A — Helm 리포지토리 방식(현행 스키마: `auth.*`)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgresql
helm search repo postgresql -o json | jq

helm install mypg bitnami/postgresql \
  --set auth.postgresPassword=postgres \
  --set auth.username=app,auth.password=app1234,auth.database=appdb \
  --set primary.persistence.enabled=false

helm list
kubectl get sts,pod,svc,ep,secret
helm show values bitnami/postgresql | sed -n '1,120p'
helm uninstall mypg
```

### 실습 B — **OCI 레지스트리** 방식(권장)

```bash
release=my-postgresql
helm install "$release" oci://registry-1.docker.io/bitnamicharts/postgresql \
  --version <원하는_차트_버전> \
  --set auth.postgresPassword=postgres \
  --set auth.username=app,auth.password=app1234,auth.database=appdb \
  --set primary.persistence.enabled=false
kubectl get sts,svc,pod -l app.kubernetes.io/instance="$release"
helm uninstall "$release"
```

### 실습 C — 구버전 값 키 예시(`postgresql.*`)

> Bitnami 차트는 버전에 따라 값 키가 **`postgresql.*` → `auth.*`**로 바뀐 이력이 있습니다. 설치 전 `helm show values`로 **현재 버전의 키**를 반드시 확인하세요.

```bash
helm install my-db \
  --set postgresql.postgresqlUsername=my-default,postgresql.postgresqlPassword=postgres,postgresql.postgresqlDatabase=mydb,postgresql.persistence.enabled=false \
  bitnami/postgresql

helm list
kubectl get sts,pod,svc,ep,secret
helm show values bitnami/postgresql | sed -n '1,120p'
helm uninstall my-db
```

---

## 5.6 Deploying a Chart with a Dependency

**왜?** “내 앱 차트가 DB 차트를 **의존성**으로 자동 설치”되면 전체 스택을 한 번에 배포할 수 있습니다.

### 실습: `music`(앱) ⇄ `postgresql`(DB) 의존

1. **차트 뼈대**

```bash
mkdir -p music/templates
cd music
```

2. **앱 Deployment/Service** — DB 연결 정보는 values에서 주입

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    {{- if .Chart.AppVersion }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name }}
    spec:
      containers:
        - image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          name: {{ .Chart.Name }}
          ports:
            - containerPort: {{ .Values.image.containerPort }}
              name: http
              protocol: TCP
          env:
            - name: QUARKUS_DATASOURCE_JDBC_URL
              value: {{ .Values.postgresql.server | default (printf "%s-postgresql" (.Release.Name)) | quote }}
            - name: QUARKUS_DATASOURCE_USERNAME
              value: {{ .Values.postgresql.postgresqlUsername | default "postgres" | quote }}
            - name: QUARKUS_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.postgresql.secretName | default (printf "%s-postgresql" (.Release.Name)) | quote }}
                  key: {{ .Values.postgresql.secretKey }}
```

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
  name: {{ .Chart.Name }}
spec:
  ports:
    - name: http
      port: {{ .Values.image.containerPort }}
      targetPort: {{ .Values.image.containerPort }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
```

3. **Chart.yaml** — 의존성 선언

```yaml
apiVersion: v2
name: music
description: A Helm chart for Music service
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: postgresql
    version: 18.0.17
    repository: "https://charts.bitnami.com/bitnami"
```

4. **values.yaml** — DB 접근 정보/시크릿 키 이름 등

```yaml
image:
  repository: quay.io/gitops-cookbook/music
  tag: "1.0.0"
  pullPolicy: Always
  containerPort: 8080

replicaCount: 1

postgresql:
  server: jdbc:postgresql://music-db-postgresql:5432/mydb
  postgresqlUsername: my-default
  postgresqlPassword: postgres
  postgresqlDatabase: mydb
  secretName: music-db-postgresql
  secretKey: postgresql-password
```

5. **의존성 다운로드 → 설치 → 점검**

```bash
helm dependency update
find charts -maxdepth 1 -type f -name 'postgresql-*.tgz' -print

helm install music-db .
# (오류 예) couldn't find key postgresql-password in Secret default/music-db-postgresql
kubectl get sts,pod,svc,ep,secret,pv,pvc

# TS 1: Secret 키/값 추가
kubectl edit secret music-db-postgresql
#   postgresql-password: cG9zdGdyZXMK

# TS 2: 앱 로그 확인(접속 실패 등 추적)
kubectl logs -l app.kubernetes.io/name=music -f
```

6. **애플리케이션 확인**

```bash
kubectl port-forward service/music 8080:8080
curl -s http://localhost:8080/song
```

7. **정리**

```bash
helm uninstall music-db
kubectl delete pvc --all
```

> **포인트**: 의존 차트의 **Secret/이름/키**가 앱 템플릿과 정확히 일치해야 합니다. 값 키가 다른(버전차) 경우 런타임에서 실패합니다.

---

## 5.7 Triggering a Rolling Update Automatically

**왜?** `ConfigMap` 내용만 바꿔도 **파드는 자동 재시작되지 않습니다**(쿠버네티스 기본 동작). 쿠버네티스는 **Deployment의 `spec.template`(파드 템플릿)**이 바뀔 때만 새 ReplicaSet을 만들고 롤링 업데이트를 시작해요. 따라서 템플릿 자체가 바뀌도록 **체크섬(sha256) 어노테이션**을 주입하는 패턴을 사용합니다.

### 핵심 아이디어

* Helm이 `configmap.yaml`(혹은 `secret.yaml`)의 **렌더 결과 전체**를 가져와 **SHA-256 해시**를 계산합니다.
* 이 해시값을 **파드 템플릿의 어노테이션**에 넣습니다.
* ConfigMap/Secret이 바뀌면 해시가 달라져 **어노테이션이 변경 → 파드 템플릿 변경으로 인식 → 자동 롤링 업데이트**가 일어납니다.

### 실습 ①: ConfigMap 변경 시 자동 롤링

```gotemplate
# templates/deployment.yaml (발췌)
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

> `$.Template.BasePath`는 현재 차트의 `templates/` 경로를 가리킵니다. `configmap.yaml`은 해당 경로에 존재해야 합니다.

**검증 절차**

```bash
# 1) values.yaml(또는 configmap.yaml 템플릿)에서 환경값 일부를 변경
# 2) 업그레이드하여 체크섬 재계산 & 어노테이션 갱신
helm upgrade pacman .

# 3) 롤아웃 상태 및 RS 변경 확인
kubectl rollout status deploy/pacman
kubectl get rs -l app.kubernetes.io/name=pacman
kubectl describe deploy pacman | sed -n '/Annotations/,+6p'
```

### 실습 ②: Secret 변경 시 자동 롤링(동일 패턴)

```gotemplate
# templates/deployment.yaml (발췌)
spec:
  template:
    metadata:
      annotations:
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```

> 비밀번호/토큰 교체 등 **Secret** 업데이트도 동일한 방식으로 자동 롤링을 유도할 수 있습니다.

### 실습 ③: 여러 파일을 각각 트래킹

```gotemplate
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```

> 파일별로 키를 분리하면, 어떤 변경이 롤아웃을 유발했는지 추적하기 쉽습니다.

### 주의 사항

* **파일 경로/이름**이 정확해야 하며, 렌더 가능한 템플릿이어야 합니다(조건부 생성 시 `include` 결과가 비어지지 않게 가드 필요).
* 미세한 공백/개행 차이도 해시에 반영됩니다(의도된 동작). 템플릿 정렬 시 일관성을 유지하세요.
* **Kustomize**는 ConfigMap/Secret **이름 자체에 해시를 붙이는 방식**(ConfigMapGenerator)으로 유사 효과를 냅니다. Helm은 **어노테이션 해시** 방식이라는 점만 다릅니다.

$1
**왜 중요한가?** 이미지 출처가 바뀌면 **보안/업데이트 경로**가 바뀝니다. 운영 중인 워크로드의 베이스 이미지를 점검해야 합니다.

* GeekNews: `docker.io/Bitnami` 삭제 안내
* Broadcom: **Bitnami Secure Images(BSI)** 공개 — 보안·라이선스 검증된 이미지 제공
* 변경 공지(2025-08-28 발효): 이후 **Legacy**는 더 이상 업데이트 없음

**BSI 테스트 예시**

```bash
docker pull bitnamisecure/nginx:latest
docker run -d -p 8080:8080 --name nginx bitnamisecure/nginx:latest
curl -s 127.0.0.1:8080
docker rm -f nginx
```

**Legacy 예시**

```bash
docker pull bitnamilegacy/nginx:1.28.0-debian-12-r4
docker run -d -p 8080:8080 --name nginx bitnamilegacy/nginx:1.28.0-debian-12-r4
curl -s 127.0.0.1:8080
docker rm -f nginx
```

> **포인트**: 신규 이미지는 BSI 네임스페이스(`bitnamisecure/*`)에서 확인하세요. Legacy는 보안 패치가 멈춥니다.

---

## OCI (OCI Registry for Helm)

**왜?** 별도 Helm repo 서버 없이 **컨테이너 레지스트리**로 차트를 표준 방식(OCI)으로 배포/인증/권한 관리할 수 있습니다.

**비교**

| 항목  | Helm repo (기존)                 | OCI (신규)                  |
| --- | ------------------------------ | ------------------------- |
| 저장소 | 정적 웹/차트 서버                     | 컨테이너 레지스트리(예: Docker Hub) |
| 배포  | `helm repo add`/`helm install` | `helm install oci://...`  |
| 인증  | repo 별도 관리                     | 레지스트리 인증 재사용              |
| 장점  | 익숙한 방식                         | 운영 단순화, 보안/권한 일원화         |
| 단점  | 서버 운영 필요                       | Helm 3.8+ 필요              |

**OCI 주소 예시**

```
oci://registry-1.docker.io/<namespace>/<chart>
oci://ghcr.io/<user>/<chart>
oci://harbor.mycompany.com/helm/<chart>
oci://us-docker.pkg.dev/<project>/helm/<chart>
```

---

## Bitnami Helm Charts (OCI 예제)

**왜?** OCI 경로로 바로 `helm pull/show/install`을 수행해 **레포 추가 없이** 차트를 다룰 수 있습니다.

```bash
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 22.0.11
helm show readme oci://registry-1.docker.io/bitnamicharts/nginx
helm show values oci://registry-1.docker.io/bitnamicharts/nginx
helm show chart  oci://registry-1.docker.io/bitnamicharts/nginx
helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx --version 22.0.11
helm get manifest my-nginx | grep 'image:'
helm uninstall my-nginx
```

---

## 6장 Cloud Native CI/CD

> **그림으로 이해하기**: Tekton은 **Step(컨테이너)**들이 모여 **Task(파드)**가 되고, Task들이 모여 **Pipeline**이 됩니다. 이벤트가 오면 **Triggers**로 파이프라인을 시작합니다. 실행 이력은 **TaskRun/PipelineRun**으로 남습니다.

### 6.1 Install Tekton

**왜?** 쿠버네티스 네이티브 CI/CD 엔진으로, Git 이벤트 → 빌드 → 이미지 푸시 → 배포 같은 흐름을 **CRD**로 선언/실행합니다.

**최신 설치**

```bash
# Pipelines
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl get crd
kubectl get all -n tekton-pipelines
kubectl get all -n tekton-pipelines-resolvers

# Triggers
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
kubectl get crd | grep triggers
kubectl get all -n tekton-pipelines

# Dashboard
kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml
kubectl get deploy -n tekton-pipelines
kubectl get svc -n tekton-pipelines tekton-dashboard -o yaml | kubectl neat | yq

# NodePort로 노출(9097 → 30000)
kubectl patch svc -n tekton-pipelines tekton-dashboard \
  -p '{"spec":{"type":"NodePort","ports":[{"port":9097,"targetPort":9097,"nodePort":30000}]}}'
# 접속: http://localhost:30000
```

**CLI 설치**

```bash
# macOS
brew install tektoncd-cli

tkn version
```

> **포인트**: 버전을 섞어 설치하면 CRD 호환성 이슈가 납니다. 한 묶음(최신/이전)을 **맞춰** 설치하세요.

---

### 6.2 Create a Hello World Task

**왜?** Tekton의 최소 실행 단위인 **Task**와 **TaskRun**의 작동을 체감합니다.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello World"
EOF

tkn task start --showlog hello
kubectl describe pod -l tekton.dev/task=hello
kubectl logs -l tekton.dev/task=hello -c step-echo
```

정리

```bash
kubectl delete taskruns --all
```

> **포인트**: 하나의 Task는 파드로 실행되고, 각 **step**은 컨테이너로 실행됩니다.

---

### 6.3 Create a Task to Compile and Package an App from Git

**왜?** 파이프라인에서 **입력/출력/작업공간**을 명확히 정의해 **재사용 가능한 단계**를 만듭니다.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: clone-read
spec:
  description: |
    This pipeline clones a git repo, then echoes the README file to the stout.
  params:
  - name: repo-url
    type: string
    description: The git repo URL to clone from.
  workspaces:
  - name: shared-data
    description: This workspace contains the cloned repo files.
  tasks:
  - name: fetch-source
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: shared-data
    params:
    - name: url
      value: $(params.repo-url)
EOF

cat <<'EOF' | kubectl create -f -
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: clone-read-run-
spec:
  pipelineRef:
    name: clone-read
  taskRunTemplate:
    podTemplate:
      securityContext:
        fsGroup: 65532
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
  params:
  - name: repo-url
    value: https://github.com/tektoncd/website
EOF

# 없으면 설치: tkn hub install task git-clone
```

정리

```bash
kubectl delete pipelineruns.tekton.dev --all
```

> **포인트**: `workspaces`는 Task 간 **파일 공유**에 쓰입니다. PVC를 자동 생성해 연결합니다.

---

### 6.4 Create a Task to Compile and Package an App from Private Git

**왜?** 사설 저장소 접근에는 **인증(SSH/토큰)**과 이를 **ServiceAccount**에 연결하는 과정이 필요합니다.

```bash
# SSH 시크릿 & ServiceAccount
SSHPK=$(base64 -w0 < ~/.ssh/id_ed25519)
SSHKH=$(ssh-keyscan github.com | grep ecdsa-sha2-nistp256 | base64 -w0)

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
data:
  id_rsa: $SSHPK
  known_hosts: $SSHKH
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: git-credentials
EOF
```

파이프라인/태스크/실행

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: my-clone-read
spec:
  params:
  - name: repo-url
    type: string
  workspaces:
  - name: shared-data
  - name: git-credentials
  tasks:
  - name: fetch-source
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: shared-data
    - name: ssh-directory
      workspace: git-credentials
    params:
    - name: url
      value: $(params.repo-url)
  - name: show-readme
    runAfter: ["fetch-source"]
    taskRef:
      name: show-readme
    workspaces:
    - name: source
      workspace: shared-data
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: show-readme
spec:
  description: Read and display README file.
  workspaces:
  - name: source
  steps:
  - name: read
    image: alpine:latest
    script: |
      #!/usr/bin/env sh
      cat $(workspaces.source.path)/readme.md
EOF

cat <<'EOF' | kubectl create -f -
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: clone-read-run-
spec:
  pipelineRef:
    name: my-clone-read
  taskRunTemplate:
    serviceAccountName: build-bot
    podTemplate:
      securityContext:
        fsGroup: 65532
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
  - name: git-credentials
    secret:
      secretName: git-credentials
  params:
  - name: repo-url
    value: git@github.com:YOUR-ID/my-sample-app.git
EOF
```

정리

```bash
kubectl delete taskruns,pipelineruns.tekton.dev --all
```

> **포인트**: `taskRunTemplate.serviceAccountName`로 런타임에 **자격 증명**을 주입합니다.

---

### 6.5 Containerize an Application Using a Tekton Task and Buildah (Kaniko 예시)

**왜?** 소스 코드를 받아 **컨테이너 이미지를 빌드 & 레지스트리에 푸시**하는 자동화를 파이프라인으로 구성합니다.

1. **Kaniko Task 설치 & Docker 자격증명 준비**

```bash
tkn hub install task kaniko

# (WSL/Linux) Docker config.json → Base64
DSH=$(cat ~/.docker/config.json | base64 -w0)

# Secret 생성
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: docker-credentials
data:
  config.json: $DSH
EOF

# ServiceAccount에 Secret 연결
kubectl create sa build-sa
kubectl patch sa build-sa -p '{"secrets": [{"name": "docker-credentials"}]}'
```

2. **Pipeline 정의 & 실행** — Git Clone → Kaniko Build & Push

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: clone-build-push
spec:
  params:
  - name: repo-url
    type: string
  - name: image-reference
    type: string
  workspaces:
  - name: shared-data
  - name: docker-credentials
  tasks:
  - name: fetch-source
    taskRef: { name: git-clone }
    workspaces:
    - name: output
      workspace: shared-data
    params:
    - name: url
      value: $(params.repo-url)
  - name: build-push
    runAfter: ["fetch-source"]
    taskRef: { name: kaniko }
    workspaces:
    - name: source
      workspace: shared-data
    - name: dockerconfig
      workspace: docker-credentials
    params:
    - name: IMAGE
      value: $(params.image-reference)
EOF

cat <<'EOF' | kubectl create -f -
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: clone-build-push-run-
spec:
  pipelineRef: { name: clone-build-push }
  taskRunTemplate:
    serviceAccountName: build-sa
    podTemplate:
      securityContext:
        fsGroup: 65532
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
  - name: docker-credentials
    secret:
      secretName: docker-credentials
  params:
  - name: repo-url
    value: https://github.com/gasida/docsy-example.git
  - name: image-reference
    value: docker.io/YOUR-ID/docsy:1.0.0
EOF
```

정리

```bash
kubectl delete taskruns,pipelineruns.tekton.dev --all
```

> **포인트**: Kaniko는 데몬 없이 이미지를 빌드/푸시합니다. 비공개 레포는 `docker-credentials`로 인증하세요.

---

## 자주 하는 실수 & 베스트 프랙티스

* `toYaml | nindent` 들여쓰기 레벨 불일치 → YAML 파싱 에러. 설치 전 `helm template`로 렌더 확인.
* 값 타입(문자/숫자/객체) 불일치 → `values.yaml` 스키마 일관성 유지.
* 차트 버전(`version`) vs 앱 버전(`appVersion`) 구분.
* 의존성 차트는 `helm dependency update`로 **잠그기**(`Chart.lock`) + 팀 표준 버전 핀.
* ConfigMap/Secret 변경은 체크섬 어노테이션으로 자동 롤링.
* Tekton **workspaces**는 파일 공유, **params**는 동적 값 주입, **ServiceAccount**는 인증 주입.

---

## 정리

* Helm: 템플릿 재사용(`_helpers.tpl`), 의존성, 릴리스 이력/롤백, OCI 배포까지 **운영형 패턴** 정리.
* Tekton: Git → Build → Push **파이프라인 표준경로**와 인증/자격증명 연계를 실습으로 체득.

---

## Cleanup

```bash
helm uninstall pacman || true
helm uninstall mypg || true
kind delete cluster --name myk8s || true
kubectl delete taskruns,pipelineruns.tekton.dev --all --ignore-not-found
```
