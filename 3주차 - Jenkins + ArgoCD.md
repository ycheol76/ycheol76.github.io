# 3주차 - Jenkins + ArgoCD

## 목차

* 전체 흐름
* Jenkins 개요 & 기본 배포 순서
* 파이프라인 핵심 개념 요약
* 파이프라인 기본 예제
* 필수 플러그인 & 자격증명
* Jenkins CI: Docker Build & Push (VERSION 파일 사용)
* Kubernetes 배포 (timeserver 예시)
* Gogs Webhook
* Jenkins: SCM 기반 Item (Jenkinsfile)
* Jenkins 컨테이너에 kubectl/Helm 설치 & kubeconfig 사용
* Blue/Green 배포 (echo-server)
* Argo CD + Helm (nginx)
* 트러블슈팅 체크
* 테스트 리포트 수집(Reports) 전략

  * JUnit/TestNG/PyTest XML 리포트
  * 커버리지(JaCoCo/Cobertura)
  * 정적분석·품질 게이트(선택)
  * E2E(UI) 테스트 수집
  * K8s 배포 검증 리포트
* Helm Values 분리 전략

  * 디렉터리 구조 예시
  * 분리 원칙
  * ConfigMap 체크섬(롤링 업데이트 보장)
  *  버전과 이미지 태그 전략
  *  Argo CD와의 연계
  * 시크릿 관리 권장
* Argo Rollouts로 Canary/BlueGreen

  * 설치(요약)
  * Canary 예시
  * BlueGreen 예시
  * 분석 템플릿(프로메테우스)
  * Jenkins/Argo CD 연계 패턴
  * 마이그레이션 팁
* 부록) 체크리스트
* 업데이트 이력

---

## 1. 전체 흐름

```
개발자 Push → Gogs → (Webhook) → Jenkins CI
  └ 빌드/테스트/도커 이미지 Push → Docker Hub
      └ (옵션) Jenkins가 kubectl/Helm으로 K8s 배포(CD)   ← Blue/Green 가능
          └ (옵션) Argo CD: Git(Helm 차트) 변경 감지 → K8s 자동 동기화
              └ (선택) Argo Rollouts로 Canary/BlueGreen 고급 전략
```

---

## 2. Jenkins 개요 & 기본 배포 순서

1. 최신 코드 가져오기 → 중앙 리포지토리에서 Fetch/Checkout
2. 단위 테스트 구현/실행 → TDD 권장
3. 코드 개발 → 실패 테스트를 통과시키도록 구현
4. 테스트 재실행 → 전부 통과 확인
5. 코드 Push & Merge → 중앙 저장소로 병합
6. 컴파일/빌드 → 병합된 코드 기준 전체 빌드
7. 통합 테스트 → 개별+통합 테스트 실행
8. 아티팩트/이미지 배포 → 빌드 산출물/도커 이미지 배포
9. E2E 테스트 → Selenium/Cypress 등으로 사용자 플로우 검증

Jenkins는 Java 서블릿 컨테이너(Tomcat 등)에서 동작, Jenkinsfile DSL로 파이프라인을 코드로 관리하며 단계 실패 시 이후 단계는 중단됩니다.

---

## 3. 파이프라인 핵심 개념 요약

* **Pipeline**: 전체 CI/CD 프로세스를 코드로 정의
* **node/agent**: 실행 노드
* **stages/stage**: 단계 묶음/개별 단계
* **steps**: 단계 내 단일 작업(쉘·빌드 등)
* **post**: 빌드 후 처리(always/success/failure/unstable/changed)
* **directive**: `environment`, `parameters`, `triggers`, `input`, `when` 등

### 파이프라인 유형

* **Pipeline script**(UI 직접 작성)
* **Pipeline script from SCM**(리포지토리의 Jenkinsfile 활용)
* **Blue Ocean**(UI 구성 → Jenkinsfile 자동 생성)

---

## 4. 파이프라인 기본 예제

### 1) 선언형

```groovy
pipeline {
  agent any
  stages {
    stage('Build')  { steps { /* build steps */ } }
    stage('Test')   { steps { /* test steps  */ } }
    stage('Deploy') { steps { /* deploy */     } }
  }
}
```

### 2) 스크립티드

````groovy
node {
  stage('Build')  { /* build steps */ }
  stage('Test')   { /* test steps  */ }
  stage('Deploy') { /* deploy */     }
}
``;

### (C) Hello / 환경변수 / 파라미터 / post
```groovy
// Hello
pipeline {
  agent any
  stages {
    stage('Hello'){ steps { echo 'Hello World' } }
    stage('Deploy'){ steps { echo 'Deployed successfully!' } }
  }
}

// 환경변수
pipeline {
  agent any
  environment { CC = 'clang' }
  stages {
    stage('Example') {
      environment { AN_ACCESS_KEY = 'abcdefg' }
      steps {
        echo "${CC}"
        sh 'echo ${AN_ACCESS_KEY}'
      }
    }
  }
}

// 파라미터
pipeline {
  agent any
  parameters {
    string(name:'PERSON',defaultValue:'Mr Jenkins',description:'Who to greet?')
    text(name:'BIOGRAPHY', defaultValue:'', description:'About the person')
    booleanParam(name:'TOGGLE', defaultValue:true, description:'Toggle')
    choice(name:'CHOICE', choices:['One','Two','Three'], description:'Pick')
    password(name:'PASSWORD', defaultValue:'SECRET', description:'Password')
  }
  stages {
    stage('Example'){
      steps {
        echo "Hello ${params.PERSON}"
        echo "Biography: ${params.BIOGRAPHY}"
        echo "Toggle: ${params.TOGGLE}"
        echo "Choice: ${params.CHOICE}"
        echo "Password: ${params.PASSWORD}"
      }
    }
  }
}

// post 사용
pipeline {
  agent any
  stages {
    stage('Compile'){ steps { echo "Compiled successfully!" } }
    stage('JUnit'){ steps { echo "JUnit passed successfully!" } }
    stage('Code Analysis'){ steps { echo "Code Analysis completed!" } }
    stage('Deploy'){ steps { echo "Deployed successfully!" } }
  }
  post {
    always   { echo "This will always run" }
    success  { echo "Run on success" }
    failure  { echo "Run on failure" }
    unstable { echo "Run on unstable" }
    changed  { echo "Run if state changed" }
  }
}
````

---

## 4. 필수 플러그인 & 자격증명

* **Plugins**: Pipeline: Stage View, Docker Pipeline, Gogs
* **Credentials (Jenkins 관리 → Credentials → Globals)**

  * `gogs-crd` (Kind: Username with password) → Gogs 토큰 사용
  * `dockerhub-crd` (Kind: Username with password) → Docker Hub 비밀번호/토큰

---

## 5. Jenkins CI: Docker Build & Push 

```groovy
pipeline {
  agent any
  environment { DOCKER_IMAGE = 'elven404/dev-app' } // 예시
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'http://192.168.45.251:3000/devops/dev-app.git',
            credentialsId: 'gogs-crd'
      }
    }
    stage('Read VERSION') {
      steps {
        script {
          def version = readFile('VERSION').trim()
          echo "Version found: ${version}"
          env.DOCKER_TAG = version
        }
      }
    }
    stage('Docker Build and Push') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-crd') {
            def appImage = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
            appImage.push()
            appImage.push('latest')
          }
        }
      }
    }
  }
  post {
    success { echo "Pushed: ${DOCKER_IMAGE}:${DOCKER_TAG}" }
    failure { echo "Pipeline failed. Check logs." }
  }
}
```

> Jenkins 컨테이너 사용 시 Docker 빌드는 Docker-in-Docker 또는 호스트 `/var/run/docker.sock` 마운트 등 구성이 필요합니다.

---

## 6. Kubernetes 배포 (timeserver 예시)

### 1) 레지스트리 Pull Secret

```bash
DHUSER=<docker_hub_username>
DHPASS=<docker_hub_password_or_token>

kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username="$DHUSER" \
  --docker-password="$DHPASS"

# 확인
kubectl get secret dockerhub-secret -o jsonpath='{.data.\\.dockerconfigjson}' | base64 -d | jq
```

### 2) Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timeserver
spec:
  replicas: 2
  selector:
    matchLabels:
      pod: timeserver-pod
  template:
    metadata:
      labels:
        pod: timeserver-pod
    spec:
      containers:
      - name: timeserver-container
        image: docker.io/elven404/dev-app:0.0.1  # 환경에 맞게
        livenessProbe:
          initialDelaySeconds: 30
          periodSeconds: 30
          httpGet: { path: /healthz, port: 80, scheme: HTTP }
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
      imagePullSecrets:
      - name: dockerhub-secret
```

### 3) Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: timeserver
spec:
  selector: { pod: timeserver-pod }
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30000
  type: NodePort
```

---

## 7. Gogs Webhook

`/data/gogs/conf/app.ini` 편집 후 컨테이너 재시작

```ini
[security]
INSTALL_LOCK = true
SECRET_KEY = j2xaUPQcbAEwpIu
LOCAL_NETWORK_ALLOWLIST = 192.168.45.251  # 각자 IP
```

Gogs 리포지토리 Webhook → Jenkins/Argo CD 로 트리거.

---

## 8. Jenkins: SCM 기반 Item (Jenkinsfile)

```groovy
pipeline {
  agent any
  environment { DOCKER_IMAGE = '<도커허브계정>/dev-app' }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'http://<자신의IP>:3000/devops/dev-app.git',
            credentialsId: 'gogs-crd'
      }
    }
    stage('Read VERSION') {
      steps {
        script {
          def version = readFile('VERSION').trim()
          echo "Version found: ${version}"
          env.DOCKER_TAG = version
        }
      }
    }
    stage('Docker Build and Push') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-crd') {
            def appImage = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
            appImage.push()
            appImage.push('latest')
          }
        }
      }
    }
  }
  post {
    success { echo "Image pushed: ${DOCKER_IMAGE}:${DOCKER_TAG}" }
    failure { echo "Pipeline failed" }
  }
}
```

---

## 9. Jenkins 컨테이너에 kubectl/Helm 설치 & kubeconfig 사용

```bash
docker compose exec --privileged -u root jenkins bash

# (아키텍처에 맞게 하나 선택)
curl -LO "https://dl.k8s.io/release/v1.32.8/bin/linux/amd64/kubectl"
# 또는
curl -LO "https://dl.k8s.io/release/v1.32.8/bin/linux/arm64/kubectl"

install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client=true

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
exit

docker compose exec jenkins kubectl version --client=true
docker compose exec jenkins helm version
```

Jenkins → Credentials(Secret file) 로 kubeconfig 업로드(ID: `k8s-crd`).

```groovy
pipeline {
  agent any
  environment { KUBECONFIG = credentials('k8s-crd') }
  stages {
    stage('List Pods') {
      steps { sh 'kubectl get pods -A --kubeconfig "$KUBECONFIG"' }
    }
  }
}
```

---

## 10. Blue/Green 배포 (echo-server)

### 매니페스트

```yaml
# deploy/echo-server-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: echo-server-blue }
spec:
  replicas: 2
  selector: { matchLabels: { app: echo-server, version: blue } }
  template:
    metadata: { labels: { app: echo-server, version: blue } }
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo
        args: ["-text=Hello from Blue"]
        ports: [{ containerPort: 5678 }]
---
# deploy/echo-server-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: echo-server-green }
spec:
  replicas: 2
  selector: { matchLabels: { app: echo-server, version: green } }
  template:
    metadata: { labels: { app: echo-server, version: green } }
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo
        args: ["-text=Hello from Green"]
        ports: [{ containerPort: 5678 }]
---
# deploy/echo-server-service.yaml
apiVersion: v1
kind: Service
metadata: { name: echo-server-service }
spec:
  selector: { app: echo-server, version: blue }  # 초기엔 blue
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
    nodePort: 30000
  type: NodePort
```

### Jenkins 파이프라인 (승인/전환/롤백)

```groovy
pipeline {
  agent any
  environment { KUBECONFIG = credentials('k8s-crd') }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'http://192.168.45.251:3000/devops/dev-app.git',
            credentialsId: 'gogs-crd'
      }
    }
    stage('container image build')  { steps { echo 'container image build' } }
    stage('container image upload') { steps { echo 'container image upload' } }
    stage('k8s deploy blue') {
      steps {
        sh "kubectl apply -f ./deploy/echo-server-blue.yaml --kubeconfig $KUBECONFIG"
        sh "kubectl apply -f ./deploy/echo-server-service.yaml --kubeconfig $KUBECONFIG"
      }
    }
    stage('approve green') { steps { input message:'Approve green version?', ok:'Yes' } }
    stage('k8s deploy green') { steps { sh "kubectl apply -f ./deploy/echo-server-green.yaml --kubeconfig $KUBECONFIG" } }
    stage('switch to green?') {
      steps {
        script {
          def ret = input message:'Green switching?', ok:'Yes', parameters:[booleanParam(defaultValue:true, name:'IS_SWITCHED')]
          if (ret) {
            sh """
              kubectl patch svc echo-server-service \
                -p '{"spec": {"selector": {"version": "green"}}}' \
                --kubeconfig $KUBECONFIG
            """
          }
        }
      }
    }
    stage('Blue cleanup/rollback') {
      steps {
        script {
          def ret = input message:'Blue cleanup or rollback?', parameters:[choice(choices:['done','rollback'], name:'ACTION')]
          if (ret == 'done') {
            sh "kubectl delete -f ./deploy/echo-server-blue.yaml --kubeconfig $KUBECONFIG"
          }
          if (ret == 'rollback') {
            sh """
              kubectl patch svc echo-server-service \
                -p '{"spec": {"selector": {"version": "blue"}}}' \
                --kubeconfig $KUBECONFIG
            """
          }
        }
      }
    }
  }
}
```

---

## 11. Argo CD + Helm (nginx)

### 차트 파일들

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}
data:
  index.html: |
{{ .Values.indexHtml | indent 4 }}
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: nginx
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        ports:
        - containerPort: 80
        volumeMounts:
        - name: index-html
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: index-html
        configMap:
          name: {{ .Release.Name }}
```

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30000
  type: NodePort
```

```yaml
# values-dev.yaml
indexHtml: |
  <!DOCTYPE html>
  <html><head><title>Welcome to Nginx!</title></head>
  <body>
    <h1>Hello, Kubernetes!</h1>
    <p>DEV : Nginx version $VERSION</p>
  </body></html>
image:
  repository: nginx
  tag: $VERSION
replicaCount: 1
```

```yaml
# values-prd.yaml
indexHtml: |
  <!DOCTYPE html>
  <html><head><title>Welcome to Nginx!</title></head>
  <body>
    <h1>Hello, Kubernetes!</h1>
    <p>PRD : Nginx version $VERSION</p>
  </body></html>
image:
  repository: nginx
  tag: $VERSION
replicaCount: 2
```

```yaml
# Chart.yaml
apiVersion: v2
name: nginx-chart
description: A Helm chart for deploying Nginx with custom index.html
type: application
version: 1.0.0
appVersion: "$VERSION"
```

### Argo CD 앱 (자동 동기화)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-nginx
  namespace: argocd
  finalizers: [resources-finalizer.argocd.argoproj.io]
spec:
  project: default
  source:
    repoURL: http://$MyIP:3000/devops/ops-deploy
    targetRevision: HEAD
    path: nginx-chart
    helm:
      valueFiles: [values-dev.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: dev-nginx
  syncPolicy:
    automated:
      prune: true
    syncOptions:
    - CreateNamespace=true
```

---

## 12. 트러블슈팅 체크

* ImagePull 실패 → 레지스트리/이미지/태그/시크릿(`imagePullSecrets`) 확인
* NodePort 충돌 → `kubectl get svc -A`로 사용중 포트 확인
* Jenkins Docker 빌드 → Docker 권한·소켓 마운트 또는 DinD 설정
* 브랜치명 불일치 → `main`/`master` 맞추기
* kubectl/helm 경로/권한 → Jenkins 컨테이너 내 설치 확인
* Gogs Webhook → URL/토큰/allowlist 확인(`app.ini`)

---

## 13. **테스트 리포트 수집(Reports) 전략**

테스트 결과를 가시화하고 실패 시 빠른 원인 파악. 단위/통합/E2E/품질 분석까지 Jenkins 파이프라인에서 자동 수집·보존.

### 1) JUnit·TestNG·PyTest XML 리포트

* **생성**: 각 테스트 프레임워크를 JUnit XML로 출력

  * Maven: `mvn -B -DskipTests=false test` (Surefire → `**/surefire-reports/*.xml`)
  * Gradle: `gradle test` (JUnit XML → `**/test-results/test/*.xml`)
  * PyTest: `pytest --junitxml=reports/junit/pytest.xml`
* **수집(Jenkins)**

```groovy
stage('Unit Test') {
  steps {
    sh 'mvn -B -DskipTests=false test'
  }
  post {
    always {
      junit testResults: '**/surefire-reports/*.xml', allowEmptyResults: true
      archiveArtifacts artifacts: 'target/**, reports/**', fingerprint: true
    }
  }
}
```

### 2) 커버리지(JaCoCo/Cobertura)

* **생성**: Maven(`jacoco-maven-plugin`) 또는 Gradle(JaCoCo)
* **수집**(JaCoCo 플러그인):

```groovy
post {
  always {
    jacoco execPattern: 'target/jacoco.exec',
           classPattern: 'target/classes',
           sourcePattern: 'src/main/java',
           exclusionPattern: '**/*Test*'
  }
}
```

### 3) 정적분석·품질 게이트(선택)

* **SonarQube**: `withSonarQubeEnv` + `sonar-maven-plugin`
* **Warnings NG**: `recordIssues tools: [java(), maven()]`

### 4) E2E(UI) 테스트 수집 (Selenium/Cypress)

* **Selenium**: Selenium Grid 또는 webdriver로 테스트 → JUnit XML 출력
* **Cypress**: `cypress run --reporter junit --reporter-options mochaFile=reports/junit/cypress-[hash].xml`
* **파이프라인**

```groovy
stage('E2E Test') {
  steps {
    sh 'npm ci && npm run e2e'  // 예시
  }
  post {
    always {
      junit '**/reports/junit/*.xml'
      archiveArtifacts artifacts: 'reports/**, screenshots/**, videos/**', fingerprint: true
      publishHTML(target: [
        reportDir: 'reports/html', reportFiles: 'index.html',
        reportName: 'E2E HTML Report', keepAll: true, alwaysLinkToLastBuild: true
      ])
    }
  }
}
```

### 5) K8s 배포 검증 리포트(배포 로그/상태 저장)

```groovy
stage('Post-Deploy Checks') {
  steps {
    sh '''
      set -e
      kubectl get deploy,po,svc -A -o wide > reports/k8s/objects.txt
      kubectl get events -A --sort-by=.lastTimestamp > reports/k8s/events.txt || true
      for p in $(kubectl get po -n dev-nginx -o name); do
        kubectl logs -n dev-nginx $p --all-containers=true > reports/k8s/logs_${p##*/}.log || true
      done
    '''
  }
  post {
    always { archiveArtifacts artifacts: 'reports/k8s/**', fingerprint: true }
  }
}
```

> Jenkins UI의 **Test Result**, **Coverage**, **HTML Reports**에서 확인. Slack/메일 알림 연동 시 실패/품질게이트 위반을 즉시 통지.

---

## 14. **Helm Values 분리 전략**

> 목표: 공통 설정은 DRY하게, 환경별 차이는 최소 diff로 유지. GitOps/Argo CD와 자연스럽게 맞물리도록 구성.

### 1) 디렉터리 구조 예시

```
charts/
  myapp/
    Chart.yaml
    values.yaml            # 공통(default)
    values-dev.yaml        # 개발 전용 오버레이
    values-stg.yaml        # 스테이징 전용 오버레이
    values-prd.yaml        # 운영 전용 오버레이
    values.schema.json     # (선택) 값 스키마 검증
    templates/*.yaml
```

### 2) 분리 원칙

* **values.yaml**(공통)

  * 이미지 리포지토리/기본 태그(또는 `.Chart.AppVersion`), 공통 라벨, 기본 리소스 요청, 공통 애노테이션(예: 체크섬)
  * 기본 `service`, `ingress`, `probes`, `securityContext`
* **환경 오버레이**(`values-*.yaml`)

  * `replicaCount`, `resources`, `hpa`, `pdb`, `nodeSelector/affinity/tolerations`
  * `image.tag`(릴리스 버전), `imagePullSecrets`, `service.type/ports`
  * 환경별 ConfigMap/Secret(민감정보는 SOPS/External Secrets 권장)

### 3) ConfigMap 체크섬(롤링 업데이트 보장)

```yaml
templates/deploy.yaml:
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### 4) 버전과 이미지 태그 전략

* **Chart.yaml**: `appVersion`을 애플리케이션 버전과 동기화
* 배포 시: `--set-string image.tag=$VERSION` 또는 GitOps에서 values 파일에 커밋

### 5) Argo CD와의 연계

* 환경별 **Application**을 분리하고 각기 다른 values 파일을 지정
* 또는 **App of Apps** 패턴으로 여러 앱/환경을 관리

### 6) 시크릿 관리 권장

* **SOPS + Age/GPG**로 Git 내 암호화 → Argo CD에서 Decrypt(전용 플러그인)
* 또는 **External Secrets Operator**로 외부 비밀 저장소(AWS/GCP/Vault 등) 연동

---

## 15. **Argo Rollouts로 Canary/BlueGreen**

kubernetes `Deployment` 대신 `Rollout` 리소스를 사용하여 점진 배포와 자동 검증을 수행합니다.

### 1) 설치(요약)

* Argo Rollouts 컨트롤러 배포(Helm 또는 kubectl)
* 트래픽 라우팅: NGINX Ingress/ALB/Istio 등과 연동(가중치 조정 지원)

### 2) Canary 예시(가중치 변경 + 수동/자동 승인)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 4
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
      - name: myapp
        image: myrepo/myapp:1.2.3
        ports: [{containerPort: 80}]
  strategy:
    canary:
      canaryService: myapp-canary       # 트래픽 분리 서비스(선택)
      stableService: myapp-stable       # 안정 버전 서비스
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 30
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 10m}
      # (선택) 자동 분석/중단
      analysis:
        templates:
        - templateName: error-rate-check
        args:
        - name: svc
          value: myapp-canary
```

> Ingress/Service Mesh가 가중치 트래픽을 지원해야 합니다.

### 3) BlueGreen 예시(서비스 스위치)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 4
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
      - name: myapp
        image: myrepo/myapp:1.2.3
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false   # 수동 승인 후 스위치
```

### 4) 분석 템플릿(예시: Prometheus 메트릭)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: http-5xx-rate
    interval: 60s
    successCondition: result < 0.01
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus-server.monitoring.svc.cluster.local
        query: |
          sum(rate(nginx_ingress_controller_requests{status=~"5..",service=~"{{args.svc}}"}[1m]))
          /
          sum(rate(nginx_ingress_controller_requests{service=~"{{args.svc}}"}[1m]))
```

### 5) Jenkins/Argo CD와의 연계 패턴

* **Jenkins**: 이미지 빌드·푸시 → 차트 값/매니페스트에 버전 커밋 → GitOps 트리거
* **Argo CD**: Rollout 리소스 동기화(Health 체크 지원) → 점진 배포/자동분석/자동롤백은 Rollouts가 처리
* **수동 게이트**: Jenkins `input` 단계 또는 Argo Rollouts `pause` 단계로 승인 흐름 구성

### 6) 마이그레이션 팁

* 기존 `Deployment` 스펙은 대부분 `Rollout`에 호환됨(필드명/경로만 변경)
* 서비스 두 개(stable/canary 또는 active/preview)와 Ingress 가중치 라우팅을 미리 준비

##
