## 목차
1. [1장 GitOps & 쿠버네티스](#1장-gitops--쿠버네티스)
2. [2장 Argo CD 기본](#2장-argo-cd-기본)
3. [3장 Argo CD 운영](#3장-argo-cd-운영)

---

## 1장 GitOps & 쿠버네티스

### 1.1. GitOps 개요

#### 1.1.1. GitOps가 등장한 배경

클라우드 네이티브 환경에서는 수많은 마이크로서비스와 인프라 설정이 얽혀 있어, 사람이 직접 명령을 입력하면서 운영하기가 점점 어려워졌다. 반면 애플리케이션 코드는 이미 **Git을 이용해 버전 관리, 코드 리뷰, CI/CD**를 적용하고 있었다.

이때 등장한 질문이 바로 다음과 같다.

> "애플리케이션 코드뿐 아니라 **인프라와 운영 설정도 Git으로 관리**할 수 없을까?"

이 질문에 대한 해답으로 2017년 **Weaveworks**의 **Flux** 팀이 제시한 개념이 **GitOps**이다. 이후 CNCF 산하의 **GitOps Working Group**이 벤더 중립적인 정의와 원칙을 정리하며, GitOps는 하나의 표준적인 운영 패턴으로 자리 잡았다.

#### 1.1.2. GitOps 한 줄 정의

여러 정의가 존재하지만, 공통된 핵심은 다음 문장으로 요약할 수 있다.

> **“시스템이 어떤 상태여야 하는지를 Git에 선언적으로 적어 두고, 실제 시스템을 그 상태에 맞게 자동으로 유지·운영하는 방식”**

여기서 중요한 포인트는 세 가지이다.

1. **Git**이 단일 진실 공급원(Single Source of Truth)이다.
2. 운영자는 클러스터에 직접 명령을 치기보다, **Git의 선언적 설정을 수정**한다.
3. **오퍼레이터/에이전트**가 Git을 읽고 실제 시스템을 그 상태와 일치시키도록 **자동으로 조정(Reconcile)** 한다.

## 1.2. GitOps 네 가지 원칙 (GitOps Principles)

CNCF GitOps Working Group은 GitOps를 다음 네 가지 원칙으로 설명한다. 이 원칙들을 이해하면, 다양한 도구(Argo CD, Flux 등)를 비교할 때도 기준점을 잡을 수 있다.

#### 1.2.1. Declarative – 선언적 구성

**선언적 구성**이란, 시스템이 **“어떤 상태여야 하는지(Desired State)”만을 기술하고, 그 상태에 도달하는 절차는 기술하지 않는 것**을 의미한다.

* 명령형(Imperative) 예시: “컨테이너 3개를 만들어라”, “이후에 1개를 더 늘려라”.
* 선언형(Declarative) 예시: “이 애플리케이션의 replica는 항상 3이어야 한다.”

쿠버네티스에서는 `Deployment` 매니페스트 안에 `replicas: 3`이라고만 적는다. 그러면 컨트롤러가 현재 상태를 확인하여,

* 파드가 1개뿐이면 2개를 더 만들고,
* 파드가 5개면 2개를 종료해서,
  결국 **3개라는 원하는 상태에 맞추는 작업을 자동으로 수행**한다.

#### 1.2.2. Versioned and Immutable – 버전이 제어되는 불변 저장소

GitOps에서는 **Git과 같은 VCS를 인프라 정의의 저장소**로 사용한다. 이 저장소는 다음과 같은 특징을 갖는다.

* Git 커밋 히스토리를 통해 **누가, 언제, 무엇을 변경했는지**를 추적할 수 있다.
* 문제가 생기면 특정 시점의 커밋으로 되돌리는 것만으로 **롤백**이 가능하다.
* 운영 관련 변경도 PR(Pull Request)을 통해 리뷰 후 머지하므로, **운영 변경 절차가 투명해지고 감사(audit)가 쉬워진다.**

이로써 Git은 코드뿐 아니라 **인프라와 설정에 대한 단일 진실 공급원**이 된다.

#### 1.2.3. Pulled Automatically – 자동화된 배포

GitOps의 핵심 중 하나는, **Git에 변경이 반영된 이후 운영자가 직접 클러스터에 명령을 치지 않는다는 점**이다.

* 이상적인 GitOps에서는 `kubectl apply`, `helm upgrade`와 같은 명령을 수동으로 실행하지 않는다.
* 대신 GitOps 에이전트/오퍼레이터가 Git 리포지터리를 감시하거나 주기적으로 읽어,

  * 변경된 선언적 정의를 인지하고,
  * 해당 내용을 클러스터에 자동으로 적용한다.

운영자의 역할은 “Git의 선언적 정의를 고치는 것”이고, 실제 반영은 **자동화된 도구가 담당**한다.

#### 1.2.4. Continuously Reconciled – 폐쇄 루프(Continuous Reconciliation)

마지막 원칙은 GitOps가 **일회성 배포가 아니라, 지속적으로 상태를 맞추는 컨트롤 루프**라는 점을 강조한다.

오퍼레이터는 다음의 과정을 끊임없이 반복한다.

1. Git에 저장된 **원하는 상태(Desired State)** 를 읽는다.
2. 클러스터의 **실제 상태(Live State)** 를 조회한다.
3. 두 상태의 차이를 계산한다.
4. 필요한 생성/수정/삭제를 수행해 실제 상태를 원하는 상태에 가깝게 만든다.

이 구조는 쿠버네티스 내부 컨트롤러(예: ReplicaSet 컨트롤러)가 하는 일과 본질적으로 동일하다. 다만 대상이 **Git에 있는 전체 매니페스트**로 확장되었다는 점이 GitOps의 특징이다.

### 1.3. Kubernetes와 GitOps의 관계

#### 1.3.1. CNCF와 커뮤니티 구조

쿠버네티스가 빠르게 성장할 수 있었던 이유 중 하나는, 구글이 프로젝트를 공개한 뒤 **리눅스 재단과 함께 CNCF(Cloud Native Computing Foundation)를 설립**하여 중립적인 거버넌스를 구축했기 때문이다.

CNCF는 다음과 같은 원칙을 갖는다.

* 프로젝트와 워킹 그룹의 **의사결정 과정이 공개적이고 투명**하다.
* 특정 회사가 투표권의 과반수를 갖지 못하게 하여, **특정 벤더에 치우치지 않도록 설계**한다.
* 커뮤니티의 참여 없이는 중요한 결정을 내릴 수 없다.

이러한 구조 덕분에, 쿠버네티스와 그 주변 생태계(Argo CD, Flux, Prometheus 등)가 **개방적이고 활발한 커뮤니티 기반**으로 성장할 수 있었다. GitOps 역시 이 생태계 안에서 발전한 운영 패턴이다.

#### 1.3.2. Kubernetes 아키텍처와 컨트롤 루프

쿠버네티스는 종종 **“플랫폼을 만들기 위한 플랫폼”**이라고 불린다. 사용자는 다양한 컴포넌트를 조합해 자신만의 PaaS나 플랫폼을 구성할 수 있으며, GitOps는 이 위에 얹을 수 있는 하나의 패턴이다.

쿠버네티스 아키텍처는 크게 **컨트롤 플레인(Control Plane)** 과 **데이터 플레인(Data Plane)** 으로 나뉜다.

* **컨트롤 플레인(Control Plane)** – 클러스터의 두뇌 역할

  * **API Server**: 모든 요청이 통과하는 관문으로, 쿠버네티스의 REST API를 제공한다.
  * **etcd**: 클러스터 상태 정보가 저장되는 분산 키-값 DB이다.
  * **Controller Manager**: 여러 컨트롤러를 실행하여 실제 상태를 원하는 상태에 맞게 지속적으로 조정한다.
  * **Scheduler**: 새로 생성된 파드를 어느 노드에 배치할지 결정한다.

* **데이터 플레인(Data Plane)** – 실제 애플리케이션이 실행되는 영역

  * **Node(노드)**: 파드가 실행되는 물리/가상 머신이다.
  * **Container Runtime**(예: containerd): 컨테이너를 생성하고 실행한다.
  * **kubelet**: 각 노드에서 파드를 관리하고, API 서버와 통신하는 에이전트이다.
  * **kube-proxy**: 서비스와 네트워크 추상화를 구현한다.

이 구조 전체가 이미 “원하는 상태를 선언하고, 컨트롤러가 실제 상태를 맞추는 구조”로 되어 있기 때문에, GitOps는 쿠버네티스와 매우 잘 맞는다. GitOps는 단지 **원하는 상태의 출처를 Git으로 삼고, 그 동기화를 외부 오퍼레이터에게 맡기는 패턴**이라고 볼 수 있다.

### 1.4. 컨트롤러, 오퍼레이터, Argo CD

#### 1.4.1. 컨트롤러(Controller)

컨트롤러는 쿠버네티스에서 **특정 리소스의 실제 상태를 감시하고, 원하는 상태와 일치하도록 조정하는 컴포넌트**이다.

* 대표 예: **ReplicaSet 컨트롤러**

  * ReplicaSet에 정의된 `replicas` 값(예: 3개)과 실제 파드 개수를 비교한다.
  * 파드가 부족하면 새로 만들고, 많으면 삭제하여 항상 목표 개수를 유지한다.

컨트롤러는 공통적으로 다음과 같은 루프를 가진다.

1. 원하는 상태(스펙)를 확인한다.
2. 현재 상태(실제 리소스)를 관찰한다.
3. 두 상태의 차이를 계산한다.
4. 필요한 조치를 수행한다.

이 구조가 바로 **컨트롤 루프(Feedback Loop)** 이며, GitOps의 Continuous Reconciliation과도 연결된다.

#### 1.4.2. 오퍼레이터(Operator)

오퍼레이터는 컨트롤러 패턴을 확장한 개념으로, **특정 도메인(예: 데이터베이스, 메시지 큐, 외부 시스템)에 대한 운영 지식을 코드로 담은 컨트롤러**라고 볼 수 있다.

* 일반 컨트롤러: 주로 쿠버네티스 내부 리소스만 다룬다.
* 오퍼레이터: 내부 리소스 + 외부 시스템까지 함께 관리할 수 있다.

  * 예: 데이터베이스 클러스터 생성/백업/복구 자동화, 외부 Git 리포지터리 연동 등.

#### 1.4.3. Argo CD

**Argo CD**는 GitOps를 구현한 대표적인 도구이다. Argo CD는 다음과 같이 동작한다.

1. 사용자가 **Git 리포지터리에 쿠버네티스 매니페스트를 선언적 형태로 저장**한다.
2. Argo CD는 이 리포지터리와 연동하여, 원하는 상태(Desired State)를 주기적으로 읽는다.
3. 실제 클러스터의 리소스 상태를 조회하여, Git에 선언된 상태와 비교한다.
4. 차이가 있을 경우, 이를 해소하기 위해 자동으로 `apply`에 해당하는 작업을 수행한다.

Git 리포지터리는 쿠버네티스 외부에 있는 시스템이므로, Argo CD는 단순 컨트롤러라기보다 **오퍼레이터에 가까운 구조**를 가진다. 내부적으로는 Custom Resource와 이를 감시하는 컨트롤러를 통해 구현된다.

### 1.5. 명령형 API vs 선언형 API

GitOps는 기본적으로 **선언형 API** 철학 위에 서 있다. 이를 이해하기 위해 먼저 쿠버네티스의 두 가지 사용 방식인 **명령형(Imperative)** 과 **선언형(Declarative)** 을 비교해 보자.

#### 1.5.1. 명령형(Imperative) 방식

명령형 방식은 사용자가 **“지금 이 작업을 해라”** 라고 직접 지시하는 방식이다.

예를 들어 네임스페이스를 만들 때:

```bash
kubectl create namespace test-imperative
```

또는 파일을 작성한 뒤:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: imperative-config-test
```

다음과 같이 적용한다.

```bash
kubectl create -f namespace.yaml
```

이미 존재하는 리소스를 교체할 때는 `kubectl replace`를 사용할 수 있다. 하지만 `replace`는 리소스를 **통째로 교체**하기 때문에, 중간에 다른 사람이 추가한 어노테이션/라벨 등의 변경사항이 덮어씌워져 사라질 수 있다. 즉, **사람이 절차를 직접 관리**해야 하고, 협업 시 충돌 가능성이 크다.

#### 1.5.2. 선언형(Declarative) 방식

선언형 방식에서는 “네임스페이스가 어떤 상태여야 하는지”만 기술한다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: declarative-files
  labels:
    namespace: declarative-files
```

이를 다음과 같이 적용한다.

```bash
kubectl apply -f namespace.yaml
```

여러 매니페스트를 모아 둔 폴더에 대해서는 다음과 같이 적용할 수 있다.

```bash
kubectl apply -f ./manifests/
```

`kubectl apply`는 현재 상태와 파일에 선언된 상태를 비교하여, **변경분만 계산해서 적용**한다. 사람이 일일이 변경 내용을 계산하지 않아도, 시스템이 디프(diff)를 계산해준다.

이 개념을 더 확장하면 다음과 같은 생각으로 이어진다.

> “로컬 폴더 대신 **Git 리포지터리**를 기준으로 삼고,
> 사람이 직접 `kubectl apply`를 하는 대신, **오퍼레이터가 주기적으로 Git을 읽어 자동으로 apply 해주면 어떨까?”

이 아이디어가 바로 **GitOps 오퍼레이터의 기본 원리**이다.

### 1.6. 간단한 GitOps 오퍼레이터 동작 원리

GitOps 오퍼레이터는 Argo CD, Flux처럼 복잡한 기능을 모두 갖추지 않더라도, 기본적으로 다음 세 가지 역할만 수행하면 된다.

#### 1.6.1. 오퍼레이터의 세 가지 핵심 동작

1. **Git 리포지터리 동기화**

   * 최초 실행 시: `git clone`으로 리포지터리를 클론한다.
   * 이후 주기적으로: `git pull` 또는 이에 상응하는 방식으로 최신 변경 사항을 가져온다.

2. **매니페스트 적용**

   * 클론/업데이트된 리포지터리에서 `namespace.yaml`, `deployment.yaml` 등의 매니페스트 파일을 찾는다.
   * 내부적으로 `kubectl apply -f`와 같은 동작을 수행하여 클러스터에 적용한다.

3. **주기적 반복(Continuous Reconciliation)**

   * 일정 주기(예: 5초)마다 위 과정을 반복한다.
   * Git의 선언된 상태와 클러스터의 실제 상태가 일치하는지 계속 확인하고, 차이가 있으면 수정한다.

#### 1.6.2. 예시: nginx 배포 관리

예를 들어, `basic-gitops-operator-config`라는 리포지터리에 `namespace.yaml`과 `deployment.yaml`이 있다고 하자.

* 오퍼레이터를 실행하면:

  * Git 리포지터리를 클론하고,
  * `nginx` 네임스페이스와 `nginx` 디플로이먼트를 `apply` 한다.

* 사용자가 수동으로 다음 명령으로 디플로이먼트를 삭제하더라도:

  ```bash
  kubectl delete deploy -n nginx nginx
  ```

  다음 주기(sync)에서 오퍼레이터는,

  * Git에는 여전히 `nginx` 디플로이먼트가 존재하지만,
  * 실제 클러스터에는 해당 디플로이먼트가 없다는 점을 발견한다.

  그러면 다시 `apply`를 수행하여 **삭제된 리소스를 자동으로 복구**한다. 이 과정이 바로 GitOps의 **“원하는 상태와 실제 상태를 자동으로 맞추는 폐쇄 루프”**에 해당한다.

보다 고급 도구인 Argo CD와 Flux는 여기에서 더 나아가, 멀티 클러스터 지원, UI, RBAC, 헬스 체크, 롤백 UI, 알림 연동 등 다양한 기능을 제공한다. 하지만 기본 철학은 이 단순한 오퍼레이터 패턴 위에 세워져 있다.

### 1.7. 요약

지금까지의 내용을 정리하면 다음과 같다.

1. **쿠버네티스**는 원래부터 “원하는 상태(Desired State)를 선언하고, 컨트롤러가 실제 상태(Live State)를 그에 맞게 조정하는” **컨트롤 루프 플랫폼**이다.
2. **GitOps**는 이 원하는 상태를 **Git에 선언적으로 저장**하고,

   * Git을 단일 진실 공급원으로 삼으며,
   * 오퍼레이터(예: Argo CD)가 Git과 클러스터 상태를 지속적으로 비교하여
   * 자동으로 동기화(배포/수정/복구)하는 운영 방식이다.

* 한 문장으로 표현하면:
  **“Git에 적힌 선언적 상태를 기준으로, 쿠버네티스 컨트롤 루프와 오퍼레이터를 이용해 인프라와 애플리케이션을 자동으로 운영하는 패턴이 GitOps이다.”**


## 2장 Argo CD 기본

### 2.1. Argo CD란 무엇인가?

#### 2.1.1. 배경: 환경 분리와 구성 드리프트 문제

전통적으로 애플리케이션은 다음과 같은 여러 환경으로 나누어 운영해 왔다.

* 개발(Dev)
* 테스트(Test)
* 스테이징(Staging)
* 프로덕션(Production)

쿠버네티스에서는 이러한 환경 분리를 여러 가지 방식으로 구현한다.

* 각 환경별로 **별도 쿠버네티스 클러스터**를 두는 방식
* 하나의 클러스터 안에서 **네임스페이스로 환경을 분리**하는 방식

예를 들어 하나의 클러스터 내에서 `dev`, `staging`, `prod` 네임스페이스를 만들고, 각각에 필요한 리소스(ConfigMap, Secret, Ingress 등)를 배포해 환경을 구성한다.

이런 구조에서 시간이 지나면 자연스럽게 **구성 드리프트(Configuration Drift)** 문제가 발생한다.

* 예: 개발 환경에는 최신 버전 애플리케이션과 최신 네트워크 정책이 배포되어 있지만, 스테이징/프로덕션에는 반영되지 않은 상태
* 이 경우 다른 환경을 개발 환경과 맞추려면, 운영자가 차이점을 직접 비교해 수동으로 수정해야 한다.

이를 단순화하기 위해 Helm, Kustomize, jsonnet 같은 **템플릿/패키지 매니저**를 사용하여 리소스를 재사용하고, 일종의 “단일 관리 지점”처럼 다루는 방식을 사용하기도 한다. 하지만 다음과 같은 한계가 있다.

* 어느 환경에 어느 버전을 배포했는지 **이력을 추적하기 어렵고**
* 템플릿/값 파일 조합이 많아질수록 **관리 복잡도가 급격히 증가**한다.

#### 2.1.2. GitOps 관점에서 본 Argo CD

GitOps 접근을 적용하면 상황이 달라진다.

* 깃 리포지터리(Git)에 **풀 리퀘스트와 모든 변경 이력**이 남는다.
* Git은 애플리케이션 및 인프라에 대한 **원천 소스(Source of Truth)**가 된다.

여기에 쿠버네티스의 컨트롤러와 비슷한 역할을 하는 도구가 있다면?

* Git 리포지터리의 선언적 구성을 읽고,
* 이를 클러스터에 자동으로 적용하며,
* 누군가 클러스터 리소스를 수동으로 변경해도, 다시 Git 상태에 맞게 되돌리는 도구

**Argo CD**가 바로 이런 역할을 수행하는 **GitOps 기반 쿠버네티스 CD(Continuous Delivery) 도구**이다.

#### 2.1.3. Argo CD의 특징 요약

Argo CD는 다음과 같은 특성을 가진다.

* **선언적(Declarative)**: Git에 선언된 쿠버네티스 매니페스트를 기준으로 동작
* **GitOps CD 도구**: Git을 기준으로 배포/롤백/상태 관리를 자동화
* **애플리케이션 컨트롤러**를 중심으로,

  * Git 리포지터리의 의도한 상태(Target State)와
  * 실제 클러스터 상태(Live State)를 **지속적으로 비교·동기화** 한다.

Argo Project 생태계에는 Argo CD 외에도 다음과 같은 도구들이 있다.

* **Argo CD** – GitOps 기반 CD
* **Argo Rollouts** – 카나리, 블루-그린 등 고급 배포 전략
* **Argo Events** – 이벤트 기반 워크플로우 트리거
* **Argo Workflows** – 워크플로우/배치 파이프라인 오케스트레이션

---

### 2.2. Argo CD의 활용 사례

Argo CD는 GitOps 철학을 바탕으로 다음과 같은 시나리오에서 자주 사용된다.

#### 2.2.1. 배포 자동화 (Automated Deployment)

* 깃 커밋 또는 CI 파이프라인이 완료되어 코드가 리포지터리에 반영되면,
* Argo CD 컨트롤러가 변경 사항을 감지하고,
* 클러스터를 **Git에 선언된 타깃 상태로 자동 동기화**한다.

#### 2.2.2. 관찰 가능성 (Observability)

* Web UI와 CLI를 통해 각 애플리케이션의

  * Sync 상태(정상/OutOfSync)
  * Health 상태(Healthy/Degraded/Missing 등)
    를 직관적으로 확인할 수 있다.
* **Argo CD Notifications** 엔진을 사용하면 상태 변경 시 알림(슬랙, 이메일 등)을 연동할 수 있다.

#### 2.2.3. 멀티 테넌시 (Multi-tenancy)

* RBAC 기반의 권한 관리를 통해 여러 팀/테넌트가 한 Argo CD 인스턴스를 공유하면서도

  * 서로 다른 프로젝트로 애플리케이션을 분리하고
  * 서로의 권한을 침범하지 않도록 제어할 수 있다.
* 여러 클러스터(Dev, Stage, Prod 등)에 대한 배포도 한 곳에서 관리 가능하다.

---

### 2.3. 핵심 개념과 용어 정리

#### 2.3.1. Reconciliation(조정)과 Sync

Argo CD의 핵심은 **Reconciliation(조정)** 루프이다.

* Git 리포지터리에는 애플리케이션의 **의도한 상태(Target State)** 가 저장되어 있다.
* 쿠버네티스 클러스터에는 **현재 상태(Live State)** 가 존재한다.
* Argo CD는 Git → 쿠버네티스 방향으로 조정 루프를 돌며,

  * Helm 차트나 Kustomize 등을 **쿠버네티스 YAML로 렌더링**하고,
  * 이를 클러스터 상태와 비교하여 동기화 여부를 판단한다.

이때 사용되는 주요 개념은 다음과 같다.

* **Target State (타깃 상태)**

  * Git 리포지터리에 정의된 애플리케이션의 의도한 상태
* **Live State (현재 상태)**

  * 실제 쿠버네티스 클러스터에 배포된 애플리케이션 상태
* **Sync Status (동기화 상태)**

  * 타깃 상태와 현재 상태가 일치(Synced)하는지, 차이가 있는지(OutOfSync)를 나타냄
* **Sync (동기화)**

  * 클러스터에 변경을 적용하여 애플리케이션을 타깃 상태로 맞추는 동작
* **Sync Operation Status (동기화 동작 상태)**

  * Sync 작업의 성공/실패 여부
* **Refresh (새로고침)**

  * Git의 최신 코드와 현재 상태를 다시 비교하는 과정
* **Health Status (서비스 상태)**

  * 애플리케이션 리소스가 실제로 정상 동작 중인지(Ready, Healthy 등)를 나타냄

#### 2.3.2. Application, Source Type 등 기본 용어

* **Application**

  * Argo CD에서 “하나의 배포 단위”를 나타내는 CRD(Custom Resource Definition).
  * 실제 쿠버네티스 리소스 그룹(Deployment, Service, ConfigMap 등)을 하나의 App으로 묶어 관리한다.

* **Application Source Type**

  * 애플리케이션을 구성하는 템플릿 도구 유형
  * 예: Helm, Kustomize, Jsonnet, Plain YAML 등

이러한 개념들을 통해 Argo CD는 “Git에 선언된 애플리케이션”을 쿠버네티스 클러스터에 일관성 있게 유지한다.

---

### 2.4. Argo CD 아키텍처

#### 2.4.1. 컨트롤러 기반 구조 개요

Argo CD의 주요 구성 요소는 모두 **쿠버네티스 컨트롤러 패턴**에 기반한다.

* 컨트롤러는 클러스터 또는 Git의 상태를 관찰하고,
* 필요한 경우 변경 사항을 요청하여,
* 현재 상태를 의도한 상태에 가깝게 유지한다.

#### 2.4.2. 주요 컴포넌트

###### (1) API 서버 (argocd-server)

* 역할

  * Web UI 및 CLI가 통신하는 엔드포인트
  * 다른 시스템(CI/CD, Argo Events 등)과도 API를 통해 상호작용
  * 애플리케이션 관리 및 상태 조회
  * 애플리케이션 동기화 트리거
  * Git 리포지터리 및 클러스터 관리
  * 인증/SSO, RBAC 정책 적용

###### (2) 리포지터리 서버 (argocd-repo-server)

* Git 리포지터리에 저장된 애플리케이션 매니페스트를 **로컬 캐시**로 유지한다.
* 다른 컴포넌트가 쿠버네티스 매니페스트를 필요로 할 때 Repo Server에 요청한다.
* 요청 시 필요한 정보 예시

  * 리포지터리 URL
  * Git 브랜치/커밋(Revision)
  * 애플리케이션 경로(Path)
  * 템플릿 세부 설정 (Helm values, ksonnet env 등)

###### (3) 애플리케이션 컨트롤러 (argocd-application-controller)

* 애플리케이션의 현재 상태를 지속적으로 확인하고, Git 리포지터리의 의도한 상태와 비교한다.
* 두 상태가 다를 경우 Sync를 수행하여 **현재 상태를 의도한 상태와 맞추려고 한다.**
* 사용자가 정의한 **훅(Hook)** 리소스를 배포 생명주기(PreSync, Sync, PostSync 등)에 맞게 실행한다.

#### 2.4.3. 동기화 트리거 방식

기본적으로 Argo CD는 **주기적인 폴링(기본 약 3분 간격)** 으로 Git 리포지터리를 확인하지만, 다음과 같은 방식으로 즉시 Sync를 트리거할 수 있다.

1. **Web UI**에서 “Sync” 버튼을 눌러 수동 동기화
2. **CLI** 명령 사용

   * `argocd app sync myapp`
3. **Webhook** 연동

   * GitHub, GitLab, Bitbucket, Gogs 등의 Webhook을 구성하여 커밋/PR 머지 시 Argo CD에 알림 → 즉시 동기화

---

### 2.5. Argo CD의 핵심 리소스 & 자격 증명

#### 2.5.1. Application CRD 예시

Argo CD는 실제 배포할 애플리케이션을 `Application` CRD로 표현한다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 13.2.10
  destination:
    namespace: nginx
    server: https://kubernetes.default.svc
```

* `source`: Git/Helm 등 애플리케이션의 원천
* `destination`: 어느 클러스터의 어느 네임스페이스에 배포할지

#### 2.5.2. AppProject – 애플리케이션 그룹화

`AppProject` CRD는 관련 있는 애플리케이션들을 논리적으로 그룹화하고, 다음을 제어하는 데 사용된다.

* 어떤 리포지터리에서만 소스를 가져올지(`sourceRepos`)
* 어느 클러스터/네임스페이스에만 배포할 수 있는지(`destinations`)
* 클러스터 범위 리소스 생성 허용/제한(`clusterResourceWhitelist` 등)

이를 통해 팀/환경별 프로젝트를 분리하고, 프로젝트 단위로 정책을 설정할 수 있다.

#### 2.5.3. 리포지터리 자격 증명 (Repository Credentials)

실제 환경에서는 대부분 **프라이빗 Git 리포지터리**를 사용하므로, Argo CD는 해당 리포지터리에 접근하기 위한 자격 증명이 필요하다.

* 자격 증명은 쿠버네티스 **Secret 리소스**로 정의한다.
* Secret에는 `argocd.argoproj.io/secret-type: repository` 라벨을 붙여, Argo CD가 이를 “저장소 자격 증명”으로 인식하게 한다.
* HTTPS, SSH 방식 모두 지원하며, URL과 토큰/SSH Key 등을 `stringData`에 설정한다.

#### 2.5.4. 클러스터 자격 증명 (Cluster Credentials)

Argo CD가 여러 클러스터(예: dev-cluster, prod-cluster)를 관리하려면, 대상 클러스터에 접근하기 위한 자격 증명이 필요하다.

* 마찬가지로 Secret 리소스를 사용하되,
* `argocd.argoproj.io/secret-type: cluster` 라벨을 사용한다.
* 내부에는 `server`(API 서버 URL), 인증 토큰, CA 인증서 정보 등이 포함된다.
* CLI를 사용하면 `argocd cluster add CONTEXT_NAME` 처럼 간단히 등록할 수도 있다.

---

### 2.6. Argo CD 설치 & 첫 애플리케이션 배포 (Helm + Guestbook)

#### 2-6-1. Helm으로 Argo CD 설치

실습에서는 Helm Chart를 이용해 Argo CD를 설치했다.

1. `argocd` 네임스페이스 생성
2. `argocd-values.yaml` 파일로 서버 서비스 타입(NodePort, 포트 등)을 설정
3. Helm Chart 설치

   * `helm repo add argo https://argoproj.github.io/argo-helm`
   * `helm install argocd argo/argo-cd --version 9.0.5 -f argocd-values.yaml -n argocd`
4. Pod, Service, Secret, ConfigMap, CRD 등의 생성 상태를 확인

이 과정을 통해 `applications.argoproj.io`, `appprojects.argoproj.io` 등의 CRD와 Argo CD 컴포넌트가 설치된다.

#### 2-6-2. Guestbook 예제 애플리케이션 배포

Helm 기반 Guestbook 예제를 `Application` CRD로 생성하여 자동 배포를 실습했다.

* `Application` 스펙에서 주요 옵션

  * `source.repoURL`: `https://github.com/argoproj/argocd-example-apps`
  * `source.path`: `helm-guestbook`
  * `syncPolicy.automated.enabled: true` – 자동 동기화 활성화
  * `syncPolicy.automated.prune: true` – Git에서 리소스 삭제 시 클러스터에서도 삭제
  * `syncPolicy.automated.selfHeal: true` – 클러스터에서 수동 변경 시 Git 상태로 복구
  * `syncOptions: [CreateNamespace=true]` – 대상 네임스페이스가 없으면 자동 생성

###### SelfHeal 동작 예시

1. Guestbook Service를 수동으로 `NodePort`로 변경
2. Argo CD가 이를 “Git 상태와 다른 Live 상태”로 인식
3. SelfHeal이 활성화되어 있으면, 다음 Sync에서 Service 스펙을 다시 Git 상태로 되돌림
4. 실습에서는 SelfHeal을 끈 뒤(NodePort 유지), 외부에서 `curl` 또는 브라우저로 접속을 확인했다.

이 과정은 “Git이 진실의 근원이며, 수동 변경은 결국 Git 상태에 의해 덮어쓰기 된다”는 GitOps의 특징을 잘 보여준다.

---

### 2.7. Argo CD CLI 요약

#### 2.7.1. CLI 설치 및 로그인

* macOS: `brew install argocd`
* Linux/WSL: 릴리스 바이너리를 직접 다운로드 후 `/usr/local/bin/argocd`로 설치

로그인 예시:

```bash
argocd login 127.0.0.1:30002 --plaintext
Username: admin
Password: <초기 비밀번호>
```

로그인 후에는 다음과 같은 명령으로 정보를 조회할 수 있다.

* `argocd account list`
* `argocd proj list`
* `argocd repo list`
* `argocd cluster list`
* `argocd app list`

> 실무에서는 admin 계정은 초기 설정에만 사용하고, 로컬 사용자/SSO 계정 및 토큰을 활용해 CI 시스템과 연동하는 것이 권장된다.

#### 2.7.2. CLI로 애플리케이션 생성 (주의점)

CLI로도 애플리케이션을 만들 수 있다.

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path helm-guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --values values.yaml
```

하지만 GitOps의 관점에서 보면,

* **애플리케이션 정의 자체도 Git에 저장하고 PR/머지로 관리하는 것이 이상적**이다.
* CLI로만 애플리케이션을 생성하면 Git에 선언이 남지 않아, GitOps 원칙이 약해질 수 있다.

따라서 학습/디버깅 용도로는 유용하지만, 실제 운영 환경에서는 **Application CRD를 Git에 두고 관리하는 방식**을 우선 고려해야 한다.

---

### 2.8. Argo CD Web-based Terminal 설정

Argo CD UI에서는 리소스 상세 화면에서 **Terminal 탭**을 통해 컨테이너 쉘에 접속할 수 있다. 이를 사용하려면 다음과 같은 설정이 필요하다.

1. `argocd-cm` ConfigMap에서 `exec.enabled: "true"`로 설정
2. `argocd-server`에 매핑된 ClusterRole에 `pods/exec` 리소스에 대한 `create` 권한 추가

이후 UI에서 파드를 선택하고 **TERMINAL** 버튼을 누르면, 웹 기반 터미널을 통해 컨테이너 내부 쉘에 접근할 수 있다.

---

### 2.9. Argo CD Autopilot

#### 2.9.1. 배경: “닭이 먼저냐, 달걀이 먼저냐” 문제

Argo CD 자체도 결국 **쿠버네티스 애플리케이션**이므로, GitOps 원칙에 따라 Argo CD의 설치·구성 자체를 Git으로 관리하고 싶다. 하지만 처음 Argo CD를 설치할 때는 아직 Argo CD가 없기 때문에, 다음과 같은 순환 문제가 생긴다.

> “Argo CD를 GitOps로 관리하려면 먼저 Argo CD가 있어야 하는데,
> 그 Argo CD는 또 GitOps로 설치·관리하고 싶다…”

이 문제를 해결하기 위해 등장한 도구가 **Argo CD Autopilot**이다.

#### 2.9.2. Autopilot의 역할

Argo CD Autopilot은 다음을 자동화한다.

* Argo CD 설치(부트스트랩)
* GitOps용 리포지터리 초기 구조 생성
* 프로젝트/애플리케이션/클러스터 리소스를 **선언적으로 관리**할 수 있도록 Git 디렉터리 구조 구성

Autopilot의 핵심 아이디어:

1. `argocd-autopilot repo bootstrap` 명령으로:

   * 대상 K8s 클러스터에 Argo CD를 설치하고,
   * GitOps 리포지터리 내 특정 디렉터리에 Argo CD 관련 Application 매니페스트를 커밋한다.
2. 그 이후 Argo CD는 **자기 자신(Argo CD 설치 및 설정)을 Git 기반으로 관리**한다.
3. 사용자는 Autopilot CLI로 프로젝트와 애플리케이션을 생성하면 되고, Autopilot은 이에 해당하는 매니페스트를 Git에 커밋한다.
4. 커밋이 완료되면 Argo CD가 자동으로 애플리케이션을 클러스터에 배포한다.

#### 2.9.3. 디렉터리 구조 & App of Apps 패턴

Autopilot이 구성하는 대표적인 디렉터리 구조:

* `apps/` – 실제 애플리케이션 정의
* `bootstrap/` – Argo CD 초기 설정과 클러스터 리소스
* `projects/` – 환경별 Project(AppProject) 정의

이 구조 속에서 다음과 같은 Application들이 존재한다.

* `autopilot-bootstrap`

  * GitOps 리포지터리의 `bootstrap` 디렉터리를 참조
  * 나머지 Application들을 관리하는 상위 App
* `argo-cd`

  * `bootstrap/argo-cd` 폴더를 참조
  * Argo CD 배포(및 ApplicationSet)를 직접 관리
* `root`

  * `projects` 폴더를 참조
  * 환경별 프로젝트를 관리하는 상위 App

이처럼 **Application이 다른 Application들을 생성·관리하는 구조**를 **App of Apps 패턴**이라고 부른다. 이 패턴을 사용하면 애플리케이션 그룹 전체를 선언적으로 관리할 수 있다.

#### 2.9.4. Project & Application 생성 흐름

Autopilot CLI를 사용하면 다음과 같이 Project와 Application을 쉽게 만들 수 있다.

* Project 생성:

  * `argocd-autopilot project create dev`
  * `argocd-autopilot project create prd`
* Application 생성:

  * `argocd-autopilot app create hello-world1 --app <demo-app-repo> -p dev --type kustomize`
  * `argocd-autopilot app create hello-world2 --app <demo-app-repo> -p prd --type kustomize`

이때 Autopilot은 내부적으로 Git 리포지터리에 필요한 매니페스트를 커밋하고, Argo CD는 이를 기준으로 애플리케이션을 배포한다.

---

### 2.10. 동기화 원리 – Hook, Sync Waves, Sync Windows

Argo CD의 Sync는 단순히 `kubectl apply`를 한 번 실행하는 것이 아니라, **여러 단계와 순서 제어 기능**을 가진다.

#### 2.10.1. 리소스 훅(Resource Hooks)

동기화 과정은 크게 다음 세 단계로 나뉜다.

1. **PreSync** – 본격적인 리소스 Sync 전에 실행
2. **Sync** – 실제 리소스를 생성/업데이트하는 단계
3. **PostSync** – 배포 후 검증/알림/후속 작업 등에 사용

그 외에 다음과 같은 훅도 있다.

* **SyncFail** – Sync 과정에서 실패했을 때 실행
* **PostDelete** – Application의 모든 리소스가 삭제된 이후 실행(Argo CD v2.10+)

훅은 쿠버네티스 리소스 매니페스트의 어노테이션으로 지정한다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
```

예를 들어, 데이터베이스 스키마 마이그레이션 Job을 PreSync 훅으로 지정하면, 본격적인 애플리케이션 배포 전에 마이그레이션 작업이 완료되도록 할 수 있다.

#### 2.10.2. 동기화 웨이브(Sync Waves)

동일한 훅 단계(PreSync/Sync/PostSync) 안에서도 **리소스 간 실행 순서**를 제어하고 싶을 때 **Sync Wave**를 사용한다.

* 웨이브는 정수 값(음수/양수 모두 가능)이며, 기본값은 `0`이다.
* 값이 작은 웨이브부터 순서대로 Sync가 진행된다.

어노테이션 예시:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
```

Argo CD는 Sync 시 다음 순서로 리소스를 정렬하여 처리한다.

1. 훅 어노테이션(PreSync/Sync/PostSync 등)
2. 웨이브 값(낮은 값부터)
3. 쿠버네티스 리소스 종류(네임스페이스 리소스 등)
4. 리소스 이름(사전순)

이를 통해, 예를 들어 다음과 같은 시나리오를 구성할 수 있다.

* PreSync Wave 0: DB 마이그레이션 Job
* PreSync Wave 1: 캐시 초기화 Job
* Sync Wave 0: ConfigMap/Secret
* Sync Wave 1: Deployment, Service
* PostSync Wave 0: Smoke Test Job

#### 2.10.3. 동기화 윈도(Sync Windows)

**Sync Window**를 사용하면 특정 시간대에 동기화를 허용 또는 차단할 수 있다.

* 예: 업무 시간에는 배포 금지, 야간에만 자동 배포 허용
* 정의는 `AppProject` 스펙의 `syncWindows` 필드에 선언한다.

예시:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
spec:
  syncWindows:
    - kind: allow
      schedule: '10 1 * * *'   # 크론 표현식
      duration: 1h
      applications:
        - '*-prod'
      manualSync: true
    - kind: deny
      schedule: '0 22 * * *'
      timeZone: "Europe/Amsterdam"
      duration: 1h
      namespaces:
        - default
```

CLI로는 다음과 같이 조회할 수 있다.

```bash
argocd proj windows list PROJECT
```

이를 통해 **배포 가능 시간대와 금지 시간대**를 프로젝트 단위로 제어할 수 있다.

---

### 2.11. 요약

1. **Argo CD**는 GitOps 방식으로 쿠버네티스 애플리케이션을 배포·관리하는 **선언적 CD 도구**이다.
2. Git 리포지터리의 **타깃 상태(Target State)** 와 클러스터의 **현재 상태(Live State)** 를 비교하고, 컨트롤러를 통해 **지속적으로 조정(Reconciliation)** 한다.
3. Argo CD 아키텍처는 API 서버, Repo 서버, 애플리케이션 컨트롤러 등으로 구성되며, 모두 쿠버네티스 컨트롤러 패턴 위에 있다.
4. Application, AppProject, Repository/Cluster Credentials 같은 CRD를 통해 애플리케이션, 프로젝트, 자격 증명을 **선언적으로 정의**할 수 있다.
5. Autopilot을 사용하면 Argo CD 자체 설치/구성도 GitOps로 관리할 수 있고, App of Apps 패턴을 통해 대규모 애플리케이션 그룹을 구조적으로 관리할 수 있다.
6. Hook, Sync Wave, Sync Window 기능을 활용하면 **배포 순서·단계·시간대**를 정교하게 제어할 수 있다.

## 3장 Argo CD 운영

### 3.1. 목표

* Argo CD를 **고가용성(HA) 모드**로 설치하고 운영 패턴 이해
* GitOps 방식으로 **Argo CD 자신을 관리(Self-managing)**하는 구조 익히기
* **관찰 가능성(Observability)** 관점에서 Prometheus/Grafana로 메트릭 수집·대시보드 구성
* **OOM, 부하, 장애, 재해 복구(백업/복원)** 시나리오에 대비하는 방법 정리

### 3.2. 실습 인프라 & HA 설치

#### 3.2.1. kind 클러스터 구성

* kind 클러스터 이름: `myk8s`
* 노드 구성

  * control-plane 1대 (NodePort 30000~30003 매핑)
  * worker 3대
* kube-ops-view 설치 (kube-system)

  * 서비스 타입: NodePort(30001)
  * 클러스터/노드 상태를 웹으로 시각화

#### 3.2.2. Argo CD HA 아키텍처 개념

* **핵심 포인트**

  * Controller / Repo-server / Server **다중화**
  * **Redis HA(Sentinel + HAProxy)** 로 캐시·세션 고가용성
  * Controller는 **Leader Election** 으로 1개만 Active, 나머지 Standby
  * Repo-server, Server는 **수평 확장 + Service 로드밸런싱**
  * 모두 `Application` CRD를 **watch** 하며 Git 상태를 클러스터에 동기화
* 주요 컴포넌트 역할

  | 컴포넌트타입권장 Replica역할                |                        |        |                                       |
  | --------------------------------- | ---------------------- | ------ | ------------------------------------- |
  | argocd-server                     | Deployment             | 2+     | UI & API 서버 (사용자/CI 진입점)              |
  | argocd-repo-server                | Deployment             | 2+     | Git에서 매니페스트 가져와 렌더링(Helm/Kustomize 등) |
  | argocd-application-controller     | StatefulSet            | 2+     | Git ↔ Cluster 상태 비교, 동기화 수행           |
  | argocd-redis-ha (+ haproxy)       | StatefulSet/Deployment | 3 / 1+ | Redis 마스터 선출, Failover, 캐시/세션 저장      |
  | argocd-dex-server                 | Deployment             | 1+     | OIDC/SSO 제공                           |
  | argocd-notifications-controller   | Deployment             | 1+     | 알림 처리(Slack/Webhook 등)                |
  | argocd-application-set-controller | Deployment             | 1+     | 다수 Application 자동 생성                  |

#### 3.2.3. HA 매니페스트로 설치 흐름

1. `argocd` 네임스페이스 생성
2. 공식 HA 매니페스트(`manifests/ha/install.yaml`) 적용
3. `argocd-server` 포트포워딩 후 웹 접속,
   `argocd-initial-admin-secret`에서 초기 비밀번호 조회
4. 설치에 사용한 `resources/` 폴더를 Git에 커밋 → 이후 GitOps로 관리 가능

#### 3.2.4. Kustomize 설치 시도 & OOMKilled 이슈

* `ch03/kustomize-installation` 예제로 HA 구성 시도
* 리소스 부족 환경에서 `argocd-redis-ha-haproxy`가
  **OOMKilled → CrashLoopBackOff** 발생
* 교훈

  * HA 구성은 리소스 소비가 크므로 **테스트 환경에서는 replica/resource 조정** 필요
  * `kustomize`로 설치해도 결국 동일한 HA 리소스가 생성 → 노드 스펙 고려 필수

---

### 3.3. Argo CD Self-managing (자기 자신을 GitOps로 관리)

#### 3.3.1. Argo CD를 Application으로 등록

1. Argo CD UI → **Settings → Repositories** 에 자신의 Git Repo 등록
2. Repo 내 `resources/` 폴더에 Argo CD 설치 매니페스트 저장
3. 아래와 같이 Argo CD 자신을 가리키는 Application 생성

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/gasida/my-sample-app
    path: resources
    targetRevision: main
  syncPolicy:
    automated: {}
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc

```

* 결과

  * Argo CD의 설치/설정/업그레이드가 **Argo CD 애플리케이션**으로 관리됨
  * 설정 변경은 **Git에 커밋 → PR 리뷰 → Merge → 자동 반영** 흐름으로 운영 가능

#### 3.3.2. 설정 변경 예시 – NetworkPolicy 삭제

* 현재 배포된 `argocd-*` NetworkPolicy 리소스 확인
* Git 저장소의 `resources` 안에서 NetworkPolicy 관련 매니페스트 삭제 → 커밋 & 푸시
* Argo CD UI에서

  * REFRESH → SYNC → SYNCHRONIZE 시 **PRUNE 옵션**으로 Git에 없는 리소스 삭제
* 또는 Application에 `syncPolicy.automated.prune: true` 설정 시 자동 제거

> 포인트: **Argo CD 자신의 보안/네트워크/리소스 설정까지 GitOps 대상으로 만든다**는 게 핵심.

---

### 3.4. 관찰 가능성(Observability) 기초 개념

#### 3.4.1. 모니터링 vs 관측 가능성

| 구분모니터링(Monitoring)관측 가능성(Observability) |                        |                                |
| --------------------------------------- | ---------------------- | ------------------------------ |
| 정의                                      | 특정 메트릭을 미리 정해놓고 계속 감시  | 외부 출력(로그/메트릭/트레이스)로 내부 상태를 추론  |
| 목표                                      | 장애·이상 감지 & 알림          | 원인 분석·최적화, “왜”를 답하는 것          |
| 데이터                                     | CPU, 메모리, 에러율 등 정해진 수치 | 로그, 메트릭, 트레이스, 이벤트 등 풍부한 텔레메트리 |
| 상호작용                                    | 임계값 기반 정적 알람           | 자유로운 쿼리, 탐색, Ad-hoc 분석         |

* 모니터링

  * "문제가 **발생했는지**"를 알려주는 역할
  * 예: CPU > 90% 5분 지속 → 알림
* 관측 가능성

  * "**왜** 문제가 발생했는지"를 다양한 데이터로 분석
  * 특히 마이크로서비스/분산 시스템에서 필수

#### 3.4.2. 메트릭 / 로그 / 트레이스

| 항목메트릭(Metrics)로그(Logs)추적(Tracing) |                     |                |                 |
| --------------------------------- | ------------------- | -------------- | --------------- |
| 형태                                | 숫자(time series)     | 텍스트/구조화 이벤트    | 요청 흐름(스팬·트레이스)  |
| 예                                 | CPU, 메모리, 요청 수, 에러율 | 에러 스택, 접근 로그 등 | A→B→C 서비스 호출 경로 |
| 목적                                | 상태·성능 추세, 알람        | 디버깅, 상세 원인 파악  | 병목 구간, 지연 구간 파악 |

#### 3.4.3. SLI / SLO / SLA 개념

* **SLI**: 측정 값 (예: 지난 30일 가용성 99.93%)
* **SLO**: 목표 값 (예: 가용성 99.9% 이상 유지)
* **SLA**: 고객과의 법적 계약 (예: 99.9% 미만 시 요금 일부 환불)

> 운영에서는 **SLI를 잘 선택하고, 현실적인 SLO를 잡은 뒤, SLA는 비즈니스와 협의**하는 구조.

---

### 3.5. Prometheus & kube-prometheus-stack

#### 3.5.1. Prometheus 개요

* 오픈소스 **모니터링 + 알림 툴킷**
* 특징

  * 시계열 저장소(TSDB) 기반, 라벨(key/value)로 차원 확장
  * 쿼리 언어: **PromQL**
  * 기본은 **Pull 방식**으로 `/metrics`를 긁어옴
  * Service Discovery / Static 설정 모두 지원
  * Alertmanager, Exporter, Pushgateway 등 다양한 생태계

#### 3.5.2. kube-prometheus-stack 설치

* Helm Chart: `prometheus-community/kube-prometheus-stack`
* 주요 설정

  * Prometheus 서비스: NodePort(30002)
  * Grafana 서비스: NodePort(30003), `admin/prom-operator`
  * Alertmanager / 기본 룰은 학습 편의상 비활성화
* 설치 후 구성 요소

  * **Prometheus**: 메트릭 수집 및 저장
  * **Grafana**: 대시보드 시각화
  * **node-exporter**: 노드 OS 레벨 메트릭 제공
  * **kube-state-metrics**: 쿠버네티스 오브젝트 상태를 메트릭으로
  * **Prometheus Operator CRD**

    * `Prometheus`, `ServiceMonitor`, `PodMonitor` 등을 선언적으로 관리

#### 3.5.3. Argo CD 메트릭 수집 – ServiceMonitor

* Argo CD는 각 컴포넌트별 `/metrics` 엔드포인트 제공

  * `argocd-metrics` (application-controller, 포트 8082)
  * `argocd-server-metrics` (server, 포트 8083)
  * `argocd-repo-server` (repo-server, 포트 8084)
  * 그 외 applicationset-controller, dex, notifications, redis-haproxy 등
* `ServiceMonitor` 리소스를 `monitoring` 네임스페이스에 생성

  * `metadata.labels.release: kube-prometheus-stack` 필수
  * `selector.matchLabels` 가 대상 Service의 라벨과 일치해야 함
  * `namespaceSelector.matchNames: [argocd]`
* 결과

  * Prometheus Targets에 `argocd-*` 엔드포인트가 Healthy 상태로 나타남
  * `argo_` / `argocd_` 로 시작하는 메트릭 조회 가능

#### 3.5.4. Grafana 대시보드 추가

* Argo CD 공식 예제 대시보드 JSON 사용

  * GitHub: `argoproj/argo-cd/examples/dashboard.json`
* Grafana → Import → JSON 붙여넣기
* 주로 보는 정보

  * Application 개수, Sync 상태, Health 상태
  * Controller/Repo-server/Server의 메트릭 및 에러율

---

### 3.6. 운영 시 중요한 Argo CD 메트릭

#### 3.6.1. OOMKilled & 리소스 이슈

* 실무에서 **가장 자주 보는 경고** 중 하나
* 컨테이너가 너무 많은 메모리를 사용 → **노드 OOM Killer**가 컨테이너 종료
* Argo CD에서 특히 주의할 대상

  * `argocd-repo-server`

    * 템플릿 렌더링 시 Helm/Kustomize 등을 fork/exec로 실행
      → 많이 동시 실행 시 메모리 급증
    * `--parallelismlimit` 옵션으로 동시에 실행할 수 있는 매니페스트 생성 작업 수 제한
  * `argocd-application-controller`

    * 많은 Application을 동시에 Sync할 때 메모리 사용 증가
    * `--kubectl-parallelism-limit` 로 동시 kubectl 실행 수 제한
* PromQL 예시(최근 재시작 + OOMKilled 탐지)

  * `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`
  * `kube_pod_container_status_restarts_total` 의 증가량과 조합해 경고 룰 구성
* 대응 전략

  1. Deployment/StatefulSet **replica 수 증가** → 부하 분산
  2. 컨테이너 **CPU/메모리 request/limit 상향**
  3. 병렬 처리 옵션(`parallelismlimit`, `kubectl-parallelism-limit`) **조정**

> 특히 동시 배포가 몰릴 때만 터지는 이슈라면, 메트릭과 배포 타이밍을 함께 보는 것이 중요.

---

### 3.7. 백업 & 재해 복구 (Disaster Recovery)

#### 3.7.1. Argo CD 상태 백업

* `argocd admin export -n argocd > backup.yaml`

  * Argo CD 관련 ConfigMap, Secret, Application 등 상태를 모두 YAML로 덤프
* 백업 전

  * `argocd login` 으로 API 서버에 로그인 필요
  * `argocd cluster list`, `argocd app list` 로 현재 상태 확인 가능

#### 3.7.2. 새 클러스터 준비 & HA 설치

1. `kind`로 새 클러스터 `myk8s2` 생성 (포트 31000~31003 노출)
2. 기존과 동일한 방식으로 `resources/namespace.yaml`, `resources/install.yaml` 적용 → HA Argo CD 설치
3. 새 클러스터의 Argo CD 서버 포트포워딩(8081 등) 후 접속, 초기 admin 비밀번호 확인

#### 3.7.3. 새 클러스터에서 복원(import)

1. 새 클러스터의 Argo CD에 로그인
2. `argocd admin import -n argocd - < backup.yaml`
3. 결과

   * Argo CD 설정(ConfigMap, Secret 등)과 Application이 새 클러스터에 생성
   * Git Repo 정보를 포함해 **동일한 GitOps 설정** 복구
   * 기존처럼 `guestbook` 등 애플리케이션도 자동 동기화

> 주의: admin 비밀번호도 **백업 시점 값으로 덮어써짐** → 로그인 정보 변경에 유의.
> 이 장의 핵심: **Argo CD 자체를 GitOps로 운영 + HA + 관찰 가능성 + 백업/복원**까지 구축해야 실제 운영에서 쓸 수 있는 GitOps 플랫폼이 된다.

