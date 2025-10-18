
# 컨테이너 빌드 & GitOps 실습 (Linux + kind)

**목표**: Linux 환경에서 Docker, Jib, Buildah, Buildpacks, Shipwright(+Kaniko) 등 다양한 방식으로 컨테이너 이미지를 빌드/푸시하고, Kustomize를 활용해 쿠버네티스 리소스를 선언적으로 배포하는 실습입니다.

---

## 목차

* [사전 준비](#사전-준비)
* [필수 툴 설치 (Linux)](#필수-툴-설치-linux)
* [Git 저장소 준비 & GitOps 개요](#git-저장소-준비--gitops-개요)
* [실습환경 구성: kind](#실습환경-구성-kind)
  * [kind란?](#kind란)
  * [클러스터 생성](#클러스터-생성)
  * [클러스터 상태 확인](#클러스터-상태-확인)
* [컨테이너 빌드](#컨테이너-빌드)
  * [Docker로 빌드/푸시/실행](#docker로-빌드푸시실행)
  * [Docker 없이 직접 OCI 이미지 빌드](#docker-없이-직접-oci-이미지-빌드)
  * [Jib로 Docker 없이 빌드](#jib로-docker-없이-빌드)
  * [Buildah로 빌드](#buildah로-빌드)
  * [Buildpacks로 빌드](#buildpacks로-빌드)
  * [kpack으로 Kubernetes에서 이미지 빌드](#kpack으로-kubernetes에서-이미지-빌드)
  * [Shipwright + Kaniko로 쿠버네티스 기반 빌드](#shipwright--kaniko로-쿠버네티스-기반-빌드)
* [Kustomize로 선언적 배포](#kustomize로-선언적-배포)
  * [Secret/ConfigMap 자동 생성](#secretconfigmap-자동-생성)
  * [부분 패치(Overlays)로 배포 차별화](#부분-패치overlays로-배포-차별화)
  * [이미지 이름/태그 치환](#이미지-이름태그-치환)
  * [namePrefix/nameSuffix, replacements 예시](#nameprefixnamesuffix-replacements-예시)
  * [base/overlay 개념과 디렉터리 구성](#baseoverlay-개념과-디렉터리-구성)
* [주의사항](#주의사항)

---

## 사전 준비

* **Docker Hub** 혹은 **quay.io**에 가입합니다.
* **kind** 설치 (Linux, **v0.30.0**)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind version
```

## 필수 툴 설치 (Linux)

```bash
# (공통 준비)
sudo apt update -y || sudo dnf -y check-update || true
sudo apt install -y curl git jq tar gzip ca-certificates || sudo dnf install -y curl git jq tar gzip ca-certificates

# kubectl 설치 (Linux)
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ] || [ "$ARCH" = "amd64" ]; then KARCH=amd64;
elif [ "$ARCH" = "aarch64" ] || [ "$ARCH" = "arm64" ]; then KARCH=arm64;
else KARCH=amd64; fi
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/${KARCH}/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client --output=yaml

# Helm 설치 (Linux)
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# krew 설치 (kubectl 플러그인 매니저)
(
  set -e
  cd "$(mktemp -d)"
  OS="$(uname | tr '[:upper:]' '[:lower:]')"
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/' -e 's/armv.*/arm/')"
  KREW="krew-${OS}_${ARCH}"
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz"
  tar zxvf "${KREW}.tar.gz"
  ./${KREW} install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# k9s 설치 (옵션1: Snap)
# sudo snap install k9s
# k9s 설치 (옵션2: 바이너리)
K9S_VERSION=v0.32.5  # 예시, 최신 버전 확인 권장
ARCH=$(uname -m); [ "$ARCH" = "x86_64" ] && K9S_ARCH=amd64 || K9S_ARCH=arm64
curl -LO "https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_${K9S_ARCH}.tar.gz"
tar -xzf "k9s_Linux_${K9S_ARCH}.tar.gz"
sudo mv k9s /usr/local/bin/

# kubecolor 설치 (Go가 있다면)
# go install github.com/kubecolor/kubecolor/cmd/kubecolor@latest
# sudo mv "$HOME"/go/bin/kubecolor /usr/local/bin/

# (선호 시) kubectl 별칭
# echo 'alias k=kubectl' >> ~/.bashrc && source ~/.bashrc
```

> **주의**: 본 문서의 모든 명령은 **`kubectl`** 표기를 사용합니다(별칭 `k` 미사용).

---

## Git 저장소 준비 & GitOps 개요

* Git 저장소 만들기: <https://github.com/gitops-cookbook/gitops-cookbook-sc> 를 **clone**하고 **자신의 저장소**를 만듭니다.

### GitOps란?

> Git 저장소를 단일 소스로 사용하여 **인프라를 코드(IaC)** 로 제공하는 운영 방식.

**GitOps 기반 CI/CD 단계**

* **CI (Continuous Integration)**
  * 개발자가 코드를 푸시하면 GitHub Actions / Jenkins / GitLab CI 등에서 **빌드 & 테스트** 실행
  * 이미지가 정상 빌드되면 Docker Registry(ECR, GCR, Docker Hub, quay.io 등)에 **푸시**
* **CD (Continuous Delivery/Deployment)**
  * **GitOps 저장소**(배포 설정)가 변경되면 Argo CD / Flux CD가 **모니터링**
  * 변경 감지 시 **쿠버네티스 클러스터에 동기화**

---

## 실습환경 구성: kind

### kind란?

컨테이너 안에서 **Kubernetes** 환경을 구성하여 별도 클러스터 없이도 로컬 Linux에서 컨테이너 엔진만으로 K8s를 구동할 수 있게 해줍니다. 일반적으로 vagrant, minikube보다 **간결하고 빠른** 편입니다.

> 권장 리소스: **vCPU 4**, **Memory 8GB** 이상

### 클러스터 생성

```bash
kind create cluster --name myk8s-1week --image kindest/node:v1.32.8 --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
- role: worker
EOF
```

> 메모: kind 생성 시 kubeconfig **current context**가 방금 만든 클러스터로 바뀝니다. 필요 시 `kubectl config use-context <다른컨텍스트>`로 변경하세요.

생성 후 `docker ps` 예시:

```
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                                                             NAMES
9461054e88f4   kindest/node:v1.32.8   "/usr/local/bin/entr…"   3 hours ago   Up 3 hours                                                                     myk8s-1week-worker
57e3f4f5b4bf   kindest/node:v1.32.8   "/usr/local/bin/entr…"   3 hours ago   Up 3 hours   0.0.0.0:40000-40001->40000-40001/tcp, 127.0.0.1:50253->6443/tcp   myk8s-1week-control-plane
```

### 클러스터 상태 확인

```bash
kubectl get nodes -o wide
kubectl cluster-info
```

> **포트 매핑 참고**: 위 kind 설정의 `extraPortMappings`는 **클러스터 내부 Service가 NodePort(예: 30000/30001)** 일 때 호스트에서 접근할 수 있게 해줍니다. 단순 Deployment만 만들면 매핑이 적용되지 않습니다.

---

## 컨테이너 빌드

컨테이너는 애플리케이션을 배포 목적에 맞게 패키징하는 **표준 형식**입니다. 이번 실습에서는 다양한 빌드 방식을 비교합니다.

### Docker로 빌드/푸시/실행

**레이어(layer)**, **이미지 빌드**, **푸시**로 구성되며, `Dockerfile`에 이미지 조립 명령을 작성합니다.

#### 예시 Dockerfile

`ch03/python-app/Dockerfile`
```dockerfile
# 기반 레이어가 되는 이미지 지정. UBI는 RHEL 기반이며 무료.
FROM registry.access.redhat.com/ubi8/python-39
# (장기 운영 시 UBI9 전환 고려: registry.access.redhat.com/ubi9/python-39)
ENV PORT=8080
EXPOSE 8080
WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENTRYPOINT ["python"]  # 컨테이너 내부의 앱 진입점
CMD ["app.py"]         # 컨테이너 시작 시 실행 명령
```

#### 빌드 & 푸시

```bash
git clone https://github.com/gitops-cookbook/chapters
cd chapters/chapters/ch03/python-app

MYREGISTRY=docker.io
MYUSER=<자신의-계정명>

# 컨테이너 이미지 빌드
docker build -f Dockerfile -t $MYREGISTRY/$MYUSER/pythonapp:latest .
# (캐시 미사용 시) --no-cache 옵션 사용 가능

# 이미지 목록 확인
docker images

# 레이어/히스토리/베이스 이미지 정보 확인 (선택)
docker inspect $MYREGISTRY/$MYUSER/pythonapp:latest | jq
docker history $MYREGISTRY/$MYUSER/pythonapp:latest
docker inspect registry.access.redhat.com/ubi8/python-39:latest | jq

# 레지스트리 로그인 & 푸시
docker login $MYREGISTRY
docker push $MYREGISTRY/$MYUSER/pythonapp:latest

# 실행 (데몬 모드; TTY 불필요)
docker run -d --name myweb -p 8080:8080 $MYREGISTRY/$MYUSER/pythonapp:latest

# 접속/로그 확인
curl 127.0.0.1:8080
docker logs myweb

# 정리
docker rm -f myweb
```

> * 한 번 pull한 이미지는 로컬 캐시에 저장되어 재사용됩니다.  
> * Linux(amd64/arm64) **아키텍처 차이**에 유의하세요. 멀티 아키텍처 빌드가 필요할 수 있습니다.

---

### Docker 없이 직접 OCI 이미지 빌드

**목표**: Dockerfile과 동등한 이미지를 **Docker 미사용** + **리눅스 유틸리티**로 직접 구성

```dockerfile
FROM scratch
COPY rootfs/ /
ENV PATH=/bin
ENTRYPOINT ["/bin/bash"]
```

> **실행 위치**: 리눅스 유틸 설치가 편한 **kind control-plane 컨테이너**에서 진행합니다.  
> 컨테이너 이름 예: `myk8s-1week-control-plane`

```bash
# control plane 접근
docker exec -it myk8s-1week-control-plane bash

# 필요 패키지 설치
apt update -y
apt install -y vim skopeo jq tar gzip coreutils

# Step 1: OCI 레이아웃 초기화
mkdir -p image/blobs/sha256
echo '{"imageLayoutVersion":"1.0.1"}' > image/oci-layout

# Step 2: 루트 파일시스템 준비
mkdir -p rootfs/bin rootfs/etc rootfs/lib

ARCH=$(uname -m)
echo "시스템 아키텍처: $ARCH"
if [ "$ARCH" = "x86_64" ]; then
  OCI_ARCH=amd64
  mkdir -p rootfs/lib/x86_64-linux-gnu
  cp -v /bin/bash rootfs/bin/bash
  cp -v /lib/x86_64-linux-gnu/libtinfo.so.* rootfs/lib/x86_64-linux-gnu/ || true
  cp -v /lib/x86_64-linux-gnu/libc.so.*     rootfs/lib/x86_64-linux-gnu/
  cp -v /lib64/ld-linux-x86-64.so.2         rootfs/lib/
elif [ "$ARCH" = "aarch64" ]; then
  OCI_ARCH=arm64
  mkdir -p rootfs/lib/aarch64-linux-gnu
  cp -v /bin/bash rootfs/bin/bash
  cp -v /lib/aarch64-linux-gnu/libtinfo.so.* rootfs/lib/aarch64-linux-gnu/ || true
  cp -v /lib/aarch64-linux-gnu/libc.so.*     rootfs/lib/aarch64-linux-gnu/
  cp -v /lib/ld-linux-aarch64.so.1           rootfs/lib/
else
  echo "미지원 아키텍처: $ARCH"; exit 1
fi
chmod 755 rootfs/bin/bash

# /etc 파일 생성
cat > rootfs/etc/passwd << 'EOF'
root:x:0:0:root:/root:/bin/bash
EOF
cat > rootfs/etc/group << 'EOF'
root:x:0:
EOF

# rootfs 압축
cd rootfs && tar -cf ../rootfs.tar . && cd ..
gzip -c rootfs.tar > rootfs.tar.gz

# SHA 계산
LAYER_TAR_SHA=$(sha256sum rootfs.tar | cut -d " " -f1)
LAYER_TARGZ_SHA=$(sha256sum rootfs.tar.gz | cut -d " " -f1)
echo "압축 전 SHA: $LAYER_TAR_SHA"
echo "압축 후 SHA: $LAYER_TARGZ_SHA"

# 레이어 저장
mv rootfs.tar.gz image/blobs/sha256/${LAYER_TARGZ_SHA}
rm rootfs.tar

# 이미지 config
cat > image/blobs/sha256/config.json << EOF
{
  "architecture": "$OCI_ARCH",
  "os": "linux",
  "config": {
    "Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "Cmd": ["/bin/bash"],
    "WorkingDir": "/"
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": ["sha256:${LAYER_TAR_SHA}"]
  },
  "history": [
    { "created": "$(date -u +%Y-%m-%dT%H:%M:%SZ)", "created_by": "manual OCI build" }
  ]
}
EOF
CONFIG_SHA=$(sha256sum image/blobs/sha256/config.json | cut -d " " -f1)
CONFIG_SIZE=$(stat -c%s image/blobs/sha256/config.json)
mv image/blobs/sha256/config.json image/blobs/sha256/${CONFIG_SHA}

# Manifest
LAYER_SIZE=$(stat -c%s image/blobs/sha256/${LAYER_TARGZ_SHA})
cat > image/blobs/sha256/manifest.json << EOF
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:${CONFIG_SHA}",
    "size": ${CONFIG_SIZE}
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:${LAYER_TARGZ_SHA}",
      "size": ${LAYER_SIZE}
    }
  ]
}
EOF
MANIFEST_SHA=$(sha256sum image/blobs/sha256/manifest.json | cut -d " " -f1)
MANIFEST_SIZE=$(stat -c%s image/blobs/sha256/manifest.json)
mv image/blobs/sha256/manifest.json image/blobs/sha256/${MANIFEST_SHA}

# Index
cat > image/index.json << EOF
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:${MANIFEST_SHA}",
      "size": ${MANIFEST_SIZE},
      "platform": { "architecture": "$OCI_ARCH", "os": "linux" }
    }
  ]
}
EOF

# 최종 패키징
tar -cf image.tar -C image .

# 업로드 (Docker Hub 또는 quay.io)
MYUSER=<dockerhub-계정>
skopeo login docker.io
skopeo copy oci-archive:image.tar docker://docker.io/${MYUSER}/myimage:latest

# 실행 확인 (호스트에서)
docker pull docker.io/${MYUSER}/myimage:latest
docker run --rm -it docker.io/${MYUSER}/myimage:latest bash
```

---

### Jib로 Docker 없이 빌드

> **전제**: Jib은 **JVM 기반 언어**(Java 등) 지원. Dockerfile 없이 소스에서 **직접 레이어링**하여 원격 레지스트리에 푸시.

```bash
# (옵션) kind worker에 Java 환경 구성; 로컬 Linux에 JDK/Maven이 있으면 로컬에서 바로 진행 가능
docker exec -it myk8s-1week-worker bash

apt update
mkdir -p /usr/share/man/man1
apt install -y perl-modules-5.36 openjdk-17-jdk maven git tree wget curl jq
java -version
mvn -version

git clone https://github.com/gitops-cookbook/chapters
cd /chapters/chapters/ch03/springboot-app/

mvn compile com.google.cloud.tools:jib-maven-plugin:3.4.6:build \
  -Dimage=docker.io/<docker-hub-id>/jib-example:latest \
  -Djib.to.auth.username=<docker-hub-id> \
  -Djib.to.auth.password=<docker-hub-token> \
  -Djib.from.platforms=linux/arm64   # 베이스 이미지가 해당 플랫폼 지원해야 함

# 호스트에서 실행 확인
docker run -d --name myweb2 -p 8080:8080 docker.io/<docker-hub-id>/jib-example:latest
curl -s 127.0.0.1:8080/hello | jq

# 정리
docker rm -f myweb2
```

---

### Buildah로 빌드

> **리눅스 전용**(macOS 불가). 데몬리스(daemonless) 이미지 빌더. 쿠버네티스 **Pod 내부**에서도 동작 가능(권한/커널 기능 충족 시).

```bash
# control-plane 컨테이너에서 진행
docker exec -it myk8s-1week-control-plane bash

# 컨테이너 런타임/이미지 조회 도구 (참고)
crictl ps || true
crictl images || true

# podman(+buildah) 설치
apt update
mkdir -p /usr/share/man/man1
apt install -y podman buildah
podman version
buildah version
buildah info

# 간단한 httpd 컨테이너 빌드 예시
mkdir httpd-containers && cd httpd-containers

# index.html
cat << 'EOF' > index.html
<html>
  <head><title>Cloudneta CICD Study</title></head>
  <body><h1>Hello, World!</h1></body>
</html>
EOF

# Dockerfile (EOL된 centos:latest 대신 rockylinux:9 사용)
cat << 'EOF' > Dockerfile
FROM rockylinux:9
RUN dnf -y install httpd && dnf clean all
COPY index.html /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
EOF

# 빌드 (arm64 예시)
buildah build --arch arm64 -f Dockerfile -t docker.io/<username>/gitops-website:latest

# 확인
buildah images && podman images

# 레지스트리에 푸시 (태그로 푸시하는 것을 권장)
buildah login --username <username> docker.io
buildah push docker.io/<username>/gitops-website:latest
```

---

### Buildpacks로 빌드

> Dockerfile 없이 **소스 코드만**으로 OCI 호환 이미지를 생성. 클라우드 네이티브 표준 준수(CNCF).

```bash
# Linux (x86_64/arm64)
PACK_VERSION=v0.35.1  # 예시, 최신 버전 확인 권장
curl -sSL "https://github.com/buildpacks/pack/releases/download/${PACK_VERSION}/pack-${PACK_VERSION}-linux.tgz" | sudo tar -C /usr/local/bin -xz pack
which pack
pack version
pack --help

# 예시 앱
git clone https://github.com/gitops-cookbook/chapters
cd chapters/chapters/ch03/nodejs-app/

# 추천 빌더 확인 (Detection)
pack builder suggest

# 빌드 (ARM64 예)
pack build nodejs-app --platform linux/arm64 --builder heroku/builder:24

docker images

# 실행 확인
docker run -d --name myapp --rm -p 3000:3000 nodejs-app
curl -s 127.0.0.1:3000
# => Hello Buildpacks!

docker rm -f myapp
```

---

### kpack으로 Kubernetes에서 이미지 빌드

> **kpack**은 Buildpacks를 **Kubernetes 네이티브**로 구동하는 컨트롤러 세트.

```bash
# 설치
kubectl apply -f https://github.com/buildpacks-community/kpack/releases/download/v0.17.0/release-0.17.0.yaml
kubectl get pods -n kpack

# 1) Registry Secret
kubectl create secret docker-registry tutorial-registry-credentials \
  --docker-username=user \
  --docker-password=password \
  --docker-server=https://index.docker.io/v1/ \
  --namespace default

# 2) ServiceAccount
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tutorial-service-account
  namespace: default
secrets:
- name: tutorial-registry-credentials
imagePullSecrets:
- name: tutorial-registry-credentials
EOF

# 3) ClusterStore (빌드팩 모음)
cat << 'EOF' | kubectl apply -f -
apiVersion: kpack.io/v1alpha2
kind: ClusterStore
metadata:
  name: default
spec:
  sources:
  - image: paketobuildpacks/java
  - image: paketobuildpacks/nodejs
EOF

# 4) ClusterStack (베이스 OS)
cat << 'EOF' | kubectl apply -f -
apiVersion: kpack.io/v1alpha2
kind: ClusterStack
metadata:
  name: base
spec:
  id: "io.buildpacks.stacks.jammy"
  buildImage:
    image: "paketobuildpacks/build-jammy-base"
  runImage:
    image: "paketobuildpacks/run-jammy-base"
EOF

cat << 'EOF' | kubectl apply -f -
apiVersion: kpack.io/v1alpha2
kind: ClusterLifecycle
metadata:
  name: default-lifecycle
spec:
  image: buildpacksio/lifecycle
EOF

# 5) Builder (Store+Stack 조합)
cat << 'EOF' | kubectl apply -f -
apiVersion: kpack.io/v1alpha2
kind: Builder
metadata:
  name: my-builder
  namespace: default
spec:
  serviceAccountName: tutorial-service-account
  tag: <docker-registry>/<repo>:<tag>
  stack:
    name: base
    kind: ClusterStack
  store:
    name: default
    kind: ClusterStore
  order:
  - group:
    - id: paketo-buildpacks/java
  - group:
    - id: paketo-buildpacks/nodejs
EOF

# 6) Image (무엇을 빌드할지)
cat << 'EOF' | kubectl apply -f -
apiVersion: kpack.io/v1alpha2
kind: Image
metadata:
  name: tutorial-image
  namespace: default
spec:
  tag: <docker-registry>/<repo>:<tag>
  serviceAccountName: tutorial-service-account
  builder:
    name: my-builder
    kind: Builder
  source:
    git:
      url: https://github.com/spring-projects/spring-petclinic
      revision: 3be289517d320a47bb8f359acc1d1daf0829ed0b
  build:
    env:
    - name: BP_JVM_VERSION
      value: "17"
EOF

# 빌드 로그 (kp CLI 필요)
# https://github.com/buildpacks-community/kpack-cli
# 설치 후:
# kp build logs tutorial-image -n default
kubectl get pods -n default
```

---

### Shipwright + Kaniko로 쿠버네티스 기반 빌드

> 쿠버네티스는 자체적으로 이미지 빌드 기능을 제공하지 않으며, **Tekton 기반**의 **Shipwright** 프레임워크로 Buildpacks/Buildah/Kaniko 등을 선택해 확장 가능.

```bash
# Tekton Pipeline 설치
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.70.0/release.yaml
kubectl get all -n tekton-pipelines
kubectl get all -n tekton-pipelines-resolvers

# Shipwright 설치
kubectl apply -f https://github.com/shipwright-io/build/releases/download/v0.11.0/release.yaml
kubectl get crd | grep shipwright
# builds.shipwright.io / (cluster)buildstrategies.shipwright.io 등 생성 확인

# 샘플 BuildStrategies 설치
kubectl apply -f https://github.com/shipwright-io/build/releases/download/v0.11.0/sample-strategies.yaml
kubectl get clusterbuildstrategy

# Docker Registry Secret
REGISTRY_SERVER=https://index.docker.io/v1/
REGISTRY_USER=<your_registry_user>
REGISTRY_PASSWORD=<your_registry_password>
EMAIL=<your_email>

kubectl create secret docker-registry push-secret \
 --docker-server=$REGISTRY_SERVER \
 --docker-username=$REGISTRY_USER \
 --docker-password=$REGISTRY_PASSWORD \
 --docker-email=$EMAIL

# Build 객체 (Kaniko)
cat << 'EOF' | kubectl apply -f -
apiVersion: shipwright.io/v1alpha1
kind: Build
metadata:
  name: kaniko-golang-build
spec:
  source:
    url: https://github.com/shipwright-io/sample-go
    contextDir: docker-build
  strategy:
    name: kaniko
    kind: ClusterBuildStrategy
  dockerfile: Dockerfile
  output:
    image: docker.io/$REGISTRY_USER/sample-golang:latest
    credentials:
      name: push-secret
EOF

kubectl get builds kaniko-golang-build -o yaml
kubectl get builds

# BuildRun 실행
cat << 'EOF' > buildrun-go.yaml
apiVersion: shipwright.io/v1alpha1
kind: BuildRun
metadata:
  generateName: kaniko-golang-buildrun-
spec:
  buildRef:
    name: kaniko-golang-build
EOF

kubectl apply -f buildrun-go.yaml
kubectl get pods -n default -w
```

---

## Kustomize로 선언적 배포

### Helm vs Kustomize 요약

| 항목     | Kustomize              | Helm                                |
| ------ | ---------------------- | ----------------------------------- |
| 구성 단위  | base + overlay         | Chart(템플릿 + values.yaml)            |
| 선언성    | **높음** (YAML 패치)       | 템플릿 렌더링 결과가 실제 매니페스트                |
| 변수 처리  | YAML Merge/Replacement | Go 템플릿(`{{ .Values.image.tag }}` 등) |
| 복잡도    | 단순/선언적                 | 유연하지만 복잡                            |
| CLI 통합 | `kubectl kustomize` 내장 | 별도 CLI(helm)                        |
| 활용     | GitOps, 환경별 overlay    | 앱 배포, 버전관리, rollback                |

### Secret/ConfigMap 자동 생성

```bash
mkdir kustomize-test && cd kustomize-test

# .properties 파일 → ConfigMap
cat << 'EOF' > application.properties
FOO=Bar
EOF

cat << 'EOF' > kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF

# 생성 미리보기
kubectl apply -k ./ --dry-run=client -o yaml
# 실제 생성/적용
kubectl apply -k ./
```

`.env` 파일로 생성:
```bash
cat << 'EOF' > .env
FOO=Bar
STUDY=Cicd
EOF

cat << 'EOF' > kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  envs:
  - .env
EOF

kubectl apply -k ./
```

**Deployment에서 ConfigMap 사용 예**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels: { app: my-app }
spec:
  selector:
    matchLabels: { app: my-app }
  template:
    metadata:
      labels: { app: my-app }
    spec:
      containers:
      - name: app
        image: nginx:alpine
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: example-configmap-1
```

**Secret 자동 생성 예**
```yaml
# 파일 그대로 secret 생성
secretGenerator:
- name: example-secret-1
  files:
  - password.txt

# key=value 리터럴로 생성
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
```

**generatorOptions 예시**
```yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
```
> 실제 운영에서는 해시를 비활성화(`disableNameSuffixHash: true`)하지 않는 편이 **롤아웃 자동화**에 유리합니다.

### 부분 패치(Overlays)로 배포 차별화

```bash
# base deployment
cat << 'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# replicas 패치
cat << 'EOF' > increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

# 리소스 제한 패치
cat << 'EOF' > set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
          limits:
            memory: 512Mi
EOF

# kustomization.yaml
cat << 'EOF' > kustomization.yaml
resources:
- deployment.yaml
patches:
- path: increase_replicas.yaml
- path: set_memory.yaml
EOF

kubectl apply -k ./
```

### 이미지 이름/태그 치환

```yaml
# kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: quay.io/nginx/nginx-unprivileged
  newTag: alpine
```

적용 후 배포 매니페스트에는 `image: quay.io/nginx/nginx-unprivileged:alpine`로 반영됩니다.

### namePrefix/nameSuffix, replacements 예시

```yaml
# deployment.yaml (일부)
containers:
- name: my-nginx
  image: nginx:alpine
  command: ["start", "--host", "MY_SERVICE_NAME_PLACEHOLDER"]
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```

```yaml
# kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"
resources:
- deployment.yaml
- service.yaml
replacements:
- source:
    kind: Service
    name: my-nginx
    fieldPath: metadata.name
  targets:
  - select:
      kind: Deployment
      name: my-nginx
    fieldPaths:
    - spec.template.spec.containers.0.command.2
```

Dry-run 출력에서 `dev-my-nginx-001`이 **Deployment/Service**에 공통 반영되고, 컨테이너 `command`의 host 자리도 자동 치환됩니다.

### base/overlay 개념과 디렉터리 구성

```bash
mkdir -p kustomize-test/base

# base
cat << 'EOF' > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: my-nginx }
spec:
  selector: { matchLabels: { run: my-nginx } }
  replicas: 2
  template:
    metadata: { labels: { run: my-nginx } }
    spec:
      containers:
      - name: my-nginx
        image: nginx:alpine
EOF

cat << 'EOF' > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels: { run: my-nginx }
spec:
  ports:
  - port: 80
    protocol: TCP
  selector: { run: my-nginx }
EOF

cat << 'EOF' > base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF

# dev overlay
mkdir -p dev
cat << 'EOF' > dev/kustomization.yaml
resources:
- ../base
namePrefix: dev-
EOF

# prod overlay
mkdir -p prod
cat << 'EOF' > prod/kustomization.yaml
resources:
- ../base
namePrefix: prod-
EOF

# 적용
kubectl apply -k dev/
kubectl apply -k prod/
```

> 동일한 base를 재사용하며 overlay만 바꿔 **환경별 구성**을 선언적으로 관리할 수 있습니다.

---

## 주의사항

* 예시는 환경/아키텍처에 따라 경로/이름이 달라질 수 있습니다. (`docker ps`, `kubectl get pods` 등으로 확인 후 조정)
* 예시의 `<username>`, `<docker-hub-id>`, `<repo>`, `<tag>` 등은 **본인 환경에 맞게 치환**하세요.
* Buildah는 Linux 전용입니다. macOS에서는 Podman Desktop 또는 Lima/Colima 등을 활용해 Linux VM에서 진행하세요.
* `kubectl create -k` 대신 **`kubectl apply -k`** 사용을 권장합니다(반복 적용/드리프트 관리에 안전).
* kind에서 외부 접근이 필요하면 **Service를 NodePort**로 만들고 `nodePort: 30000` 등으로 매핑한 포트를 사용하세요.
