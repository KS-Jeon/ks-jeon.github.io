---
title: "따라 배우는 쿠버네티스 필기 노트"
date: 2026-02-03 17:27:00 +0900
categories: [DevOps, Kubernetes]
tags: [kubernetes, devops, container, study-notes]
description: "따라 배우는 쿠버네티스 강의 영상의 수강 내용을 노트 필기"
toc: true
---

## Lab Environment

본 실습은 WSL 환경에서 kind(Kubernetes in Docker) 기반 Kubernetes 클러스터를 구성해 진행했으며, 강의를 수강하며 내용을 정리한 학습 요약 노트입니다.

### Tools

- [TTABAE-LEARN](https://www.youtube.com/@ttabae-learn)
- [kind](https://kind.sigs.k8s.io/)

---

## Pod

### livenessProbe

- 파드 내 **컨테이너 헬스 체크 및 자가 복구(Self-healing)** 기능
- 헬스 체크 실패 시 **컨테이너 재시작**

#### livenessProbe 메커니즘

1. **httpGet**
    - 컨테이너에 HTTP GET 요청
    - 응답 코드가 `200`이 아니면 컨테이너 재시작
2. **tcpSocket**
    - 지정된 포트로 TCP 연결 시도
    - 연결 실패 시 컨테이너 재시작
3. **exec**
    - 컨테이너 내부에 명령어 실행
    - 종료 코드가 `0`이 아니면 컨테이너 재시작

#### livenessProbe 주요 매개변수

- `periodSeconds` : 헬스 체크 반복 주기 (초)
- `initialDelaySeconds` : 파드 시작 후 헬스 체크 시작까지 대기 시간
- `timeoutSeconds` : 헬스 체크 응답 대기 시간
- `failureThreshold` : 몇 번 실패 시 비정상으로 판단할지
- `successThreshold` : 몇 번 성공 시 정상으로 판단할지

위 매개변수들은 YAML 템플릿에서 설정 가능

### Init Container

- **애플리케이션 컨테이너 실행 전에 먼저 실행되는 컨테이너**
- 모든 Init Container가 **순차적으로 완료된 후** 앱 컨테이너 실행
- 초기 설정, 사전 작업(파일 생성, 권한 설정 등)에 사용

**참고 자료**
- [초기화 컨테이너](https://kubernetes.io/ko/docs/concepts/workloads/pods/init-containers/)

### Infra Container (Pause Container)

- 파드 생성 시 함께 생성되는 **인프라용 컨테이너**
- 네트워크 네임스페이스 등 **파드의 기본 인프라 구성 담당**
- 실제 애플리케이션 로직은 수행하지 않음

### Static Pod

- 노드에 직접 접근하여 **kubelet이 관리하는 static pod 디렉터리**에 YAML 저장
- API Server를 거치지 않고 **kubelet 데몬이 직접 파드 실행**
- YAML 파일 삭제 시 파드도 자동 삭제됨

**참고 자료**
- [스태틱(static) 파드 생성하기](https://kubernetes.io/ko/docs/tasks/configure-pod-container/static-pod/)

### Pod 리소스 할당 (Requests / Limits)

- **Resource Requests**
    - 파드 실행을 위해 필요한 **최소 리소스 요청**
    - 스케줄링 기준으로 사용됨
- **Resource Limits**
    - 파드가 사용할 수 있는 **최대 리소스 제한**
- 메모리 limit 초과 시
    - 컨테이너 종료(OOMKilled)
    - 이후 다시 스케줄링됨

**참고 자료**
- [컨테이너 및 파드 CPU 리소스 할당](https://kubernetes.io/ko/docs/tasks/configure-pod-container/assign-cpu-resource/)

### Pod 환경 변수 설정

- 파드 내 컨테이너 실행 시 필요한 환경 변수
- 컨테이너 이미지 빌드 시 정의 가능

#### Dockerfile 예시 (Nginx)

```docker
ENV NGINX_VERSION 1.19.2
ENV NJS_VERSION 0.4.3
```

- 파드 실행 시 YAML을 통해 **기존 환경 변수 덮어쓰기 가능**

### Pod 구성 패턴 (Multi-Container Pod)

#### Sidecar

- 메인 애플리케이션을 **보조**하는 컨테이너
- 예) 로그 수집, 모니터링 에이전트

#### Adapter

- 애플리케이션의 출력 형식을 **표준 인터페이스로 변환**
- 예) 로그 포맷 변환

#### Ambassador

- 외부 통신을 전담하는 **프록시(대리인) 컨테이너**
- 예) DB 프록시, API 게이트웨이 역할

**참고 자료**
- [Multi-Container Pod Design Patterns - CKAD Course](https://matthewpalmer.net/kubernetes-app-developer/articles/multi-container-pod-design-patterns.html)

---

## Pod Operations Practice

### 1⃣ Static Pod 생성

#### 문제

다음 요구사항을 만족하는 **Static Pod**를 생성하시오.

#### 요구사항

- `node01`에서 `redis` 이미지를 사용하는 **`mydb`라는 이름의 Static Pod**를 생성
- 해당 Pod는 **desktop-worker(node01)에서만 실행**
- 장애 발생 시 **자동 재생성 / 재시작**되어야 함

#### 조건

- Static Pod 경로: `/etc/kubernetes/manifests`
- kubelet이 Static Pod를 사용하도록 설정되어 있어야 함
- Pod **`mydb-node01`** 이 `Running` 상태여야 함

### 풀이 절차

desktop-worker(node01)에 접속하여 `/etc/kubernetes/manifests` 경로에 YAML 템플릿 생성

```bash
docker exec -it desktop-worker bash
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: redis
  name: mydb
spec:
  containers:
  - image: redis
    name: pod
    ports:
    - containerPort: 6379
```

### 2⃣ 조건에 맞는 Pod 생성

#### 문제

다음 요구사항을 만족하는 **Pod**를 생성하시오.

#### 요구사항

- **Pod 이름**: `myweb`
- **컨테이너 이미지**: `nginx:1.14`
- **리소스 요구 사항**
    - 요청(Request)
        - CPU: `200m`
        - Memory: `500Mi`
    - 제한(Limit)
        - CPU: `1 core`
        - Memory: `1Gi`
- **애플리케이션 동작에 필요한 환경 변수**
    - `DB=mydb`

#### 조건

- Pod는 **`product` 네임스페이스**에서 실행되어야 함
- Pod 상태는 `Running` 이어야 함

### 풀이 절차

product 네임스페이스 생성 후 YAML 템플릿 실행

```bash
kubectl create namespace product
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: myweb
  namespace: product
spec:
  containers:
  - name: nginx-container
    image: nginx:1.14
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "200m"
        memory: "500Mi"
      limits:
        cpu: 1
        memory: "1Gi"
    env:
    - name: DB
      value: "mydb"
```

---

## Controller

### Controller 개요

- **API Server, etcd, Scheduler, Controller**가 협업하여 선언한 상태(desired state)와 실제 상태(actual state)를 일치시킴
- 주요 목적: **파드 개수 보장, 상태 유지, 자동 복구**

### ReplicationController (RC)

- 요구하는 **파드 개수(replicas)** 를 항상 보장
- 파드 집합의 실행 상태를 안정적으로 유지

#### 기본 구성

- `selector`
- `replicas`
- `template`

#### 특징

- 초기 컨트롤러 개념
- 현재는 **ReplicaSet / Deployment 사용 권장**

### ReplicaSet (RS)

- ReplicationController와 동일한 역할
- 파드 개수 보장
- **더 풍부한 selector 지원**

**Selector 종류**

- `matchLabels`
- `matchExpressions`

#### matchExpressions 연산자

- **In:** key + values가 일치하는 파드 선택
- **NotIn:** key는 일치, values는 불일치
- **Exists:** key 라벨이 존재하는 파드 선택
- **DoesNotExist:** key 라벨이 없는 파드 선택

#### 기타 특징

- `apiVersion: apps/v1` 필수
- 일반적으로 **Deployment가 내부적으로 관리**

### Deployment

- **ReplicaSet을 컨트롤**하여 파드 수 조절
- 무중단 배포 및 롤백 제공

#### 주요 기능

- Rolling Update
- Rolling Back

#### Rollout 설정 요약 (YAML)

- `change-cause`
    - Deployment 변경 사유 기록
    - `kubectl rollout history`로 확인
- `progressDeadlineSeconds`
    - 지정 시간 내 롤아웃 진전 없으면 실패 처리
- `revisionHistoryLimit`
    - 롤백 가능한 이전 ReplicaSet 보관 개수
- `strategy`
    - 무중단 업데이트 방식 설정
- `maxSurge`
    - 업데이트 중 추가 생성 가능한 파드 비율
- `maxUnavailable`
    - 업데이트 중 동시에 내려가도 되는 파드 비율

### DaemonSet

- **모든 노드에 파드 1개씩 실행** 보장

#### 주요 사용 사례

- 로그 수집기
- 모니터링 에이전트
- 네트워크 CNI

#### 업데이트 특징

- Rolling Update 지원
- `maxUnavailable`로 동시 업데이트 노드 수 제어
- 노드 단위 **병렬 업데이트**

### StatefulSet

- 파드의 **상태(이름, 네트워크 ID, 볼륨)** 를 유지하는 컨트롤러

#### 특징

- 각 파드에 고정된 네트워크 ID 및 전용 스토리지 보장
- 순차적 생성 / 삭제 / 스케일링 지원
- 데이터 보존이 핵심

#### 사용 사례

- DB
- 메시지 큐
- 순서 및 데이터 무결성이 중요한 워크로드

#### 업데이트 특징

- Rolling Update 지원
- **항상 순차적 업데이트 (병렬 불가)**
- `partition`으로 업데이트 대상 파드 범위 제어 가능

### Job

- 쿠버네티스는 파드를 기본적으로 **항상 Running 상태**로 유지하려 함
- **Batch 처리 파드**는 작업 완료 후 **종료됨**
- Batch 처리에 적합한 컨트롤러로 **파드의 성공적인 완료를 보장**

#### 동작 방식

- 비정상 종료 시 → **다시 실행**
- 정상 종료 시 → **Job 완료(Succeeded)**

#### API 버전

```yaml
apiVersion: batch/v1
```

#### 주요 옵션

- `completions`
    - Job이 **성공해야 하는 총 파드 실행 횟수**
- `parallelism`
    - **동시에 실행되는 파드 개수**
- `activeDeadlineSeconds`
    - Job 전체 실행 최대 시간(초), 초과 시 **Failed**
- `restartPolicy (Never / OnFailure)`
    - 파드 실패 시 재시작 방식
- `backoffLimit`
    - 파드 실패 허용 횟수, 초과 시 **Job Failed**

### CronJob

- **Job 컨트롤러를 주기적으로 실행**하기 위한 컨트롤러
- 리눅스 **cron 스케줄링 기능을 Job에 적용한 API**
- 반복 실행이 필요한 배치 작업에 사용

#### 사용 사례

- 데이터 백업
- 이메일 발송
- 주기적 태스크 초기화

#### 스케줄 예시

```yaml
schedule: "0 3 1 * *"
```

#### Cron 스케줄 필드 의미

- Minutes: `0 ~ 59`
- Hours: `0 ~ 23`
- Day of the month: `1 ~ 31`
- Month: `1 ~ 12`
- Day of the week: `0 ~ 6`

#### API 버전

```yaml
apiVersion: batch/v1
```

#### 주요 옵션

- `concurrencyPolicy`
    - CronJob 실행이 **겹칠 때 동작 방식 정의:** `Allow` / `Forbid`
- `startingDeadlineSeconds`
    - **실행 지연 허용 시간(초)**
- `successfulJobsHistoryLimit`
    - **성공한 Job 히스토리 보관 개수**

---

## Controller Operations Practice

### 1⃣ ReplicationController

#### 문제 1

다음의 조건으로 **ReplicationController**를 사용하는 `rc-lab.yaml` 파일을 생성하고 동작시키시오.

#### 요구사항

- labels: `name=apache`, `app=main`, `rel=stable`
- 컨테이너 이미지: `httpd:2.2`
- Pod 개수: `2`
- rc name: `rc-mainui`

#### 조건

- 현재 디렉토리에 `rc-lab.yaml` 파일을 생성해야 함
- 애플리케이션 동작은 **파일을 이용해 실행**

### 풀이 절차

`rc-lab.yaml`  생성 후 커맨드 실행

```bash
kubectl apply -f rc-lab.yaml
```

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-mainui
spec:
  replicas: 2
  selector:
    name: apache
    app: main
    rel: stable
  template:
    metadata:
      name: mainui-pod
      labels:
        name: apache
        app: main
        rel: stable
    spec:
      containers:
      - name: httpd-container
        image: httpd:2.2
        ports:
        - containerPort: 80
```

#### 문제 2

동작 중인 `httpd:2.2` 컨테이너를 **3개로 확장하시오.**

### 풀이 절차

```bash
kubectl scale rc rc-mainui --replicas=3
```

### 2⃣ ReplicaSet

#### 문제 1

다음의 조건으로 **ReplicaSet**을 사용하는 `rs-lab.yaml` 파일을 생성하고 동작 시키시오.

#### 요구사항

- labels: `name=apache`, `app=main`, `rel=stable`
- 컨테이너 이미지: `httpd:2.2`
- Pod 개수: `2`
- rs name: `rs-mainui`

#### 조건

- 현재 디렉토리에 `rs-lab.yaml` 파일을 생성해야 함
- 애플리케이션 동작은 **파일을 이용해 실행**

### 풀이 절차

`rs-lab.yaml` 생성 후 커맨드 실행

```bash
kubectl apply -f rs-lab.yaml
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-mainui
spec:
  replicas: 2
  selector:
    matchLabels:
      name: apache
      app: main
      rel: stable
  template:
    metadata:
      name: mainui-pod
      labels:
        name: apache
        app: main
        rel: stable
    spec:
      containers:
      - name: httpd-container
        image: httpd:2.2
        ports:
        - containerPort: 80
```

#### 문제 2

동작 중인 `httpd:2.2` 컨테이너를 **1개로 축소하시오.**

```bash
kubectl scale rs rs-mainui --replicas=1
```

### 3⃣ Deployment

#### 문제 1

다음의 조건으로 **Deployment**를 사용하는 `dep-lab.yaml` 파일을 생성하고 `apply` 명령으로 동작시키시오.

#### 요구사항

- labels: `name=apache`, `app=main`, `rel=stable`
- 컨테이너 이미지: `httpd:2.2`
- Pod 개수: `2`
- deployment name: `dep-mainui`
- annotations: `kubernetes.io/change-cause: version 2.2`

#### 조건

- 현재 디렉토리에 `dep-lab.yaml` 파일을 생성해야 함
- `apply` 명령으로 실행

### 풀이 절차

`dep-lab.yaml` 생성 후 커맨드 실행

```bash
kubectl apply -f dep-lab.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-mainui
  annotations:
    kubernetes.io/change-cause: version 2.2
spec:
  replicas: 2
  selector:
    matchLabels:
      name: apache
      app: main
      rel: stable
  template:
    metadata:
      labels:
        name: apache
        app: main
        rel: stable
    spec:
      containers:
      - name: httpd-container
        image: httpd:2.2
        ports:
        - containerPort: 80
```

#### 문제 2

동작 중인 `dep-mainui`의 이미지를 `httpd:2.4` 버전으로 **rolling update** 하시오. (단, apply 명령 사용)

### 풀이 절차

`dep-lab.yaml` 변경 후 커맨드 실행

```bash
kubectl apply -f dep-lab.yaml --record
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-mainui
  annotations:
    kubernetes.io/change-cause: version 2.4
spec:
  replicas: 2
  selector:
    matchLabels:
      name: apache
      app: main
      rel: stable
  template:
    metadata:
      labels:
        name: apache
        app: main
        rel: stable
    spec:
      containers:
      - name: httpd-container
        image: httpd:2.4
        ports:
        - containerPort: 80
```

#### 문제 3

현재 `dep-mainui`의 히스토리를 확인하고 이전 버전으로 rollback 하시오.

### 풀이 절차

rollout history 명령어로 히스토리 확인 후 rollout undo 명령어로 리비전 수행

```bash
kubectl rollout history deployment dep-mainui
# 히스토리 확인 하여 리비전 수행 - ex)1, 2, 3, 4
kubectl rollout undo deployment dep-mainui --to-revision=1
```

#### 문제 4

현재 동작 중인 Pod의 **httpd 이미지 버전**을 확인하시오.

### 풀이 절차

describe 명령어로 이미지 버전 확인

```bash
kubectl describe pod <파드명>
```

---

## Service

### Service 개요

- 동일한 기능을 수행하는 **Pod 그룹의 단일 진입점(Endpoint)** 제공
- Pod는 동적으로 생성/삭제되므로 **고정 IP가 없음**
- Service는 **Virtual IP를 제공**하여 안정적인 접근 보장
- 내부적으로 **로드밸런싱 수행**

### ClusterIP

- 기본 Service 타입
- 클러스터 내부 통신용
- Pod 그룹에 대한 **Virtual IP 생성**
- 외부에서는 접근 불가

#### 특징

- Selector로 선택된 Pod에 로드밸런싱
- 내부 서비스 간 통신에 사용

### NodePort

- 외부에서 접근 가능한 포트를 **모든 노드에 개방**
- 내부적으로 ClusterIP 생성 후 NodePort 할당

#### 특징

- 기본 포트 범위: `30000–32767`
- 외부에서 `NodeIP:NodePort`로 접근

#### 동작흐름

```
External Client → NodeIP:NodePort → ClusterIP → Pod
```

### LoadBalancer

- 클라우드 환경에서 **외부 로드밸런서 자동 생성**
- 클라우드 컨트롤러가 로드밸런서 프로비저닝 수행

#### 특징

- 내부적으로 NodePort 자동 생성
- 퍼블릭 IP 또는 DNS로 외부 접근 가능
- 외부에서 `NodeIP:NodePort`로 접근

#### 동작흐름

```
External Client → Cloud LoadBalancer → NodePort → Pod
```

### ExternalName

- 외부 도메인을 **클러스터 내부 DNS로 매핑**
- Service나 Pod 없이 **DNS alias 역할**

#### 사용 사례

- 외부 DB 연결
- SaaS API 호출

### Headless Service

- ClusterIP가 없는 Service (`clusterIP: None`)
- 단일 진입점 없이 **Pod 각각에 직접 접근**

#### 특징

- DNS가 각 Pod IP를 반환
- 주로 StatefulSet에서 사용

### kube-proxy

- Kubernetes Service의 실제 네트워크 동작 담당
- Service와 Pod 간 트래픽 전달 처리

#### 역할

- Service → Pod 연결
- iptables 또는 IPVS 규칙 생성
- NodePort 트래픽 처리

---

## Ingress

### Ingress

- HTTP/HTTPS 기반으로 클러스터 내부 서비스를 외부에 노출
- L7(애플리케이션 계층) 트래픽 제어

#### 특징

- URL 기반 라우팅
- 하나의 IP로 여러 서비스 노출 가능
- SSL 인증서 처리 가능

#### 주요 기능

- 경로 기반 라우팅 (Path-based routing)
- 도메인 기반 라우팅 (Host-based routing)
- TLS 종료(SSL termination)

#### 동작흐름

```
External Client → Ingress Controller → Service → Pod
```

---

## Label & Annotation

### Label

#### 개요

- 노드를 포함하여 **Pod, Deployment 등 모든 Kubernetes 리소스에 할당 가능**
- 리소스의 특성을 분류하고 **Selector를 통해 선택**할 때 사용
- **Key-Value 쌍** 형태로 구성
    - 예: `name=mainui`, `rel=stable`

#### 주요 특징

- 리소스를 **논리적으로 그룹화**하는 용도
- Service, Deployment, ReplicaSet 등이 **Pod를 선택할 때 사용**
- 필터링 및 스케줄링 기준으로 활용

#### 공식 문서

**참고 자료**
- [레이블과 셀렉터](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels/)

### Node Label

#### 개요

- Worker Node의 특성을 Label로 설정
- 하드웨어 또는 역할 기반 분류에 사용

#### 예시

- GPU 노드: `gpu=true`
- SSD 노드: `disk=ssd`
- 역할 기반: `role=frontend`

#### 활용 목적

- 특정 노드에만 Pod를 배치
- `nodeSelector`, `nodeAffinity`에서 사용

### Annotation

#### 개요

- Label과 동일하게 **Key-Value 형식**
- 하지만 **리소스 선택(Selector)에는 사용되지 않음**
- **메타데이터 기록 목적**으로 사용

#### 주요 특징

- Kubernetes 내부 동작이나 외부 시스템과 연동 시 사용
- 사람이 읽거나 도구가 해석하는 정보 저장

#### 사용 예시

- Deployment의 **rolling update 기록**
- 릴리즈 버전 정보
- 로깅 및 모니터링 설정
- 빌드 정보, 작성자, Git commit 등

### Label을 이용한 Canary 업데이트

#### 주요 업데이트 방식

- **Blue/Green 업데이트**
- **Canary 업데이트**
- **Rolling 업데이트**

#### Canary 업데이트 개념

- 기존 버전을 유지한 상태에서
- **일부 Pod만 신규 버전으로 배포**
- 신규 버전의 안정성 확인 후 전체 전환

#### Label 기반 Canary 예시

```yaml
# 기존 버전
labels:
  app: mainui
  rel: stable

# 신규 버전(카나리)
labels:
  app: mainui
  rel: canary
```

Service는 `app=mainui`만 선택하도록 설정하면 트래픽이 분산되어 카나리 테스트 가능

---

## ConfigMap & Secret

### ConfigMap

#### 개념

- 컨테이너 구성 정보를 한 곳에 모아서 관리
- 애플리케이션 설정값을 코드와 분리
- 환경 변수, 설정 파일, 매개변수 등으로 전달 가능

#### 주요 특징

- 민감하지 않은 일반 설정 데이터 저장
- Key-Value 구조
- 여러 Pod에서 공유 가능
- 런타임에 설정 주입 가능

#### 데이터 전달 방식

- **특정 Key만 적용 (`configMapKeyRef`)**

```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DB_HOST
```

- **전체 Key 적용 (`configMapRef`)**

```yaml
envFrom:
- configMapRef:
    name: app-config
```

#### 생성 명령어

- **리터럴 값으로 생성**

```bash
kubectl create configmap my-config \
  --from-literal=key1=value1
```

- **파일 기반 생성**

```bash
kubectl create configmap my-config \
  --from-file=config.txt
```

### Secret

#### 개념

- 패스워드, 인증 토큰, SSH 키 등 민감한 정보 저장
- 보안 데이터 전용 리소스

#### 주요 특징

- Base64 인코딩 저장
- etcd에 저장됨
- 환경 변수, 볼륨, CLI 인자로 전달 가능
- 민감한 데이터는 Secret 사용
- 일반 설정은 ConfigMap 사용

#### 데이터 전달 방식

- **환경 변수 사용**

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

- **볼륨 마운트 사용**

```yaml
volumes:
- name: secret-vol
  secret:
    secretName: db-secret
```

#### Secret 생성 명령어 및 타입

- **Secret 생성 명령어**

```bash
kubectl create secret generic db-secret \
  --from-literal=password=1234
```

- **Secret 타입**

| 타입 | 설명 |
| --- | --- |
| generic | 일반 Secret |
| docker-registry | 도커 레지스트리 인증 정보 |
| tls | TLS 인증서 |

---

## Pod Scheduling

### Node Selector

- Worker Node에 할당된 Label을 이용해 Node를 선택
- nodeSelector: 명시된 모든 Label이 포함되어야 배치됨

#### Node Label 설정

```bash
kubectl label nodes <노드명> <레이블 키>=<레이블 값>
kubectl label nodes lab-worker gpu=true
kubectl get nodes -L gpu
```

### Affinity & AntiAffinity

- nodeAffinity: 요구 조건을 명시하여 특정 노드에만 Pod가 실행되도록 유도

#### nodeAffinity 요구 조건

- 엄격한 요구: `requiredDuringSchedulingIgnoredDuringExecution`
- 선호도 요구: `preferredDuringSchedulingIgnoredDuringExecution`

#### podAffinity

- 특정 Pod와 가까운 위치에 배치

#### podAntiAffinity

- 특정 Pod와 멀리 떨어지도록 배치

#### podAffinity 요구 조건

- 엄격한 요구: `requiredDuringSchedulingIgnoredDuringExecution`
- 선호도 요구: `preferredDuringSchedulingIgnoredDuringExecution`

#### topologyKey

- 노드 Label을 이용해 pod의 affinity와 anti-affinity를 설정하는 기준
- 예: `kubernetes.io/hostname`, `topology.kubernetes.io/zone`

**스케줄링 흐름**

1. Pod Label 기준으로 대상 노드 후보 선택
2. topologyKey를 기준으로 실제 배치 위치 결정

### Taint & Toleration

- Node에는 **taint**, Pod에는 **toleration** 설정
- Worker Node에 taint가 설정된 경우 동일 값을 가진 toleration이 있는 Pod만 배치

#### 동작 개념

- toleration이 없는 Pod → taint가 있는 Node에 배치 불가
- toleration이 있는 Pod → 해당 taint Node에도 배치 가능

#### effect 필드 종류

- `NoSchedule` → toleration이 없으면 배치되지 않음
- `PreferNoSchedule` → 가능하면 배치하지 않음 (강제 아님)
- `NoExecute` → toleration이 없으면 기존 Pod도 종료됨

### Cordon & Drain

#### cordon

- Node 스케줄링 중단 및 허용 기능
- 특정 노드에 Pod 스케줄을 금지하거나 해제

```bash
kubectl cordon <노드명>
kubectl uncordon <노드명>
```

#### drain

- 특정 노드에서 동작 중인 모든 Pod를 안전하게 제거

```bash
kubectl drain <노드명> [옵션]
```

주요 옵션

- `--ignore-daemonsets` → DaemonSet Pod는 무시
- `--force` → RC, RS, Job, StatefulSet 등으로 관리되지 않는 단독 Pod까지 제거

---

## Pod Scheduling Operations Practice

### 1⃣ 다음과 같은 Node Label을 설정하시오.

#### 조건

```
node1: disktype=std, gpu=true
node2: disktype=ssd, gpu=false
node3: disktype=ssd, gpu=true
```

### 풀이 절차

```bash
kubectl label nodes lab-worker disktype=std gpu=true
kubectl label nodes lab-worker2 disktype=ssd gpu=false
kubectl label nodes lab-worker3 disktype=ssd gpu=true
```

### 2⃣ NodeSelector를 이용하여 tensorflow 컨테이너 실행

#### 조건

```
podname: pod-ml
image: tensorflow/tensorflow:nightly-jupyter
NODE: disktype=ssd, gpu=true
```

### 풀이 절차

아래의 YAML 템플릿 생성 후 apply

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ml
  labels:
    app: tensorflow
spec:
  containers:
  - image: tensorflow/tensorflow:nightly-jupyter
    name: tensoflow-container
  nodeSelector:
    disktype: ssd
    gpu: "true"
```

### 3⃣ mongoDB 데이터베이스를 최대한 멀리 떨어뜨려 배치

#### 조건

```
name: mongodb-pod
image: mongo
containerPort: 27017
NODE: pod-ml이 동작 중인 노드와 최대한 멀리 떨어진 노드에 배치
```

### 풀이 절차

아래의 YAML 템플릿 생성 후 apply

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-pod
  labels:
    app: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb-container
    ports:
    - containerPort: 27017
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - tensorflow
        topologyKey: "kubernetes.io/hostname"
```

---

## Authentication & Authorization

### API Access Control

- API 인증 요청 주체
    
    → User / Group, Service Account (Application)
    

#### Authentication: 정말 너 맞니?

- User 또는 Application이 API 접근을 허가받는 과정
- API 서버에 요청을 보내기 전에 신원 확인 수행

**인증 방식**

- Client Certificate
- Bearer Token
- HTTP Basic Authentication

#### Authorization: 이 작업 수행할 권한 있니?

- 인증된 주체가 해당 작업을 수행할 수 있는지 확인
- RBAC (Role-Based Access Control) 모델 기반
- 요청 ID에 적절한 Role이 있는지 검사

#### Admission Control: 이 요청이 적절한가?

- 요청이 올바른 형식인지 검사
- 요청이 처리되기 전에 수정 또는 정책 적용 가능
- 보안 정책, 리소스 제한 등을 적용하는 단계

### Authentication

- API 서버에 접근하기 위해 반드시 인증 작업 필요

#### Human User / Group

- 클러스터 외부에서 쿠버네티스를 조작하는 사용자
- kubectl, API 호출 등을 통해 접근

**참고 자료**
- [Issue a Certificate for a Kubernetes API Client Using A CertificateSigningRequest](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/)

**참고 자료**
- [Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

#### Service Account

- 쿠버네티스가 내부적으로 관리하는 계정
- Pod 및 쿠버네티스 리소스의 인증 주체로 사용

**특징**

- Pod 실행 시 ServiceAccount를 명시하지 않으면 해당 Namespace의 `default` ServiceAccount 자동 할당
- 클러스터 내부 애플리케이션 인증에 사용

**참고 자료**
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)

### Authorization

- 특정 User 또는 ServiceAccount가 접근하려는 API에 대한 권한 설정
- 권한이 있는 주체만 API 접근 허용

#### Role

- 어떤 API를 이용할 수 있는지 정의
- 쿠버네티스 리소스 접근 권한을 정의
- **특정 Namespace 내에서만 유효**

**참고 자료**
- [kubectl create role](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_role/)

#### RoleBinding

- User/Group 또는 ServiceAccount와 Role을 연결
- 해당 Namespace 내에서 권한 적용

**참고 자료**
- [kubectl create rolebinding](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_rolebinding/)

#### ClusterRole / ClusterRoleBinding

- Role/RoleBinding을 **클러스터 전체 범위**로 확장한 개념
- 모든 Namespace에 동일한 권한 적용 가능

**참고 자료**
- [kubectl create clusterrole](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_clusterrole/)

**참고 자료**
- [kubectl create clusterrolebinding](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_clusterrolebinding/)

---

## Authentication & Authorization Operations Practice

### ClusterRole & ClusterRoleBinding 생성 실습

다음 조건에 맞게 ClusterRole과 ClusterRoleBinding을 생성하시오.

#### 조건

```
유저: app-manager

권한: 클러스터 내 모든 namespace에서 deployment, pod, service 리소스에 대해
create, list, get, update, delete 권한 부여
```

#### 1⃣ 유저 app-manager 인증서 생성 및 등록

```
인증서 이름: app-manager
username: app-manager
```

### 풀이 절차

openssl key 및 csr 생성

```bash
openssl genrsa -out app-manager.key 2048
openssl req -new -key app-manager.key -out app-manager.csr -subj "/CN=app-manager"
```

YAML 파일 apply 후 주석과 같이 생성 절차 진행

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: app-manager
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1d6Q0NBVU1DQVFBd0ZqRVVNQklHQTFVRUF3d0xZWEJ3TFcxaGJtRm5aWEl3Z2dFaU1BMEdDU3FHU0liMwpEUUVCQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUUROSVpJd2VURnAva2FJVDVTWXkvUjJNZDd2ckk3czFTbUdzTk12Ci9PdEVHZktkNlZHR0pqSDVQMmdHQWljMFpVTStJWWR2WU53VWxzcFlCNlg0NmY1U1ZNLzgvajNrdHg0WU9tYUQKeGgveEJUaDVyaU1PZHI2ZEJYZU13Rm1HaXQrUkxlVnJuY1MyaEZQUDY2dHk5QWlFbHFWR1RPVDA2azJNQ1gyQgpCQngwc0lLcU05bTkxT09jdEZmUDYxTmxXZElhU2poUU9BTGhiekFXVm9qODJ1OUlyaUt5amZFK3FBOTB4WjRkCk1UUHVlVnA2Vi9xd3BRWnhabXl3Y3NleGt2MEM2RWg4RzlaTnhXNDhrRHMvenN2VFFzUHFlM0pHTkp6dnhmdksKWkxEOFJ3YlZRdVp3SGxBQ3JmeE1BODVodmIySE5uOXU5UjgvZWd2QW1VR0taamlYQWdNQkFBR2dBREFOQmdrcQpoa2lHOXcwQkFRc0ZBQU9DQVFFQVBuRzhodnF0TVlQZmI1NElCM1NhOGFNV2pBMXhuM0R1OTJYNGt2VFpBazQzCnV2Q3lCQTA1bFB3Y1Vxc244U1Z3RTYxWG9KbThDRkttZURkdzh5cEdNQXgwUUFza0R0N0VQanJtOVBia09jTXcKcE10aXl1anV1V2xCNTFMUGx4TDUvaVFtSzhpakpWSHRjSVVERzhwOWNIajJZR0xybXZKMnlObWlqckF2K3dCbgpRN24xRm1SZG1DTXNqdno3Qk91Zmk0UU1pbU1ocmpUWUFJR3pnVVNhalE4MXFjYll5blpvcGF6em95UnBJSjhFCjJHelVvOHkwU3d1d1pvR2tQOGRiZ1VXdjJKVE5HVGo1N3JsNFllVXlmdGVEYUNjaEx1MUdsSHVLa2JMYmNNWjgKRmNpcnRuSWJDcHMyNVB0bWFoL2JYZjF2TDR0WkFYZzEzaHdwNFYzam53PT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
# 1. csr 생성 후 approve
# kubectl certificate approve app-manager
# 2. crt 추출
# kubectl get csr app-manager -o jsonpath='{.status.certificate}' | base64 -d > app-manager.crt
# 3. user 생성
# kubectl config set-credentials app-manager --client-key=app-manager.key --client-certificate=app-manager.crt --embed-certs=true
# 3. context 생성
# kubectl config set-context app-manager --cluster=kind-lab --user=app-manager
```

#### 2⃣ ClusterRole 생성

```
name: app-access
```

### 풀이 절차

YAML 파일 생성 후 apply

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-access
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  verbs:
  - create
  - list
  - get
  - update
  - delete
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - create
  - list
  - get
  - update
  - delete
```

#### 3⃣ ClusterRoleBinding 생성

```
name: app-binding-manager
user: app-manager
role: app-access
```

### 풀이 절차

YAML 파일 생성 후 apply

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: app-binding-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: app-access
subjects:
- kind: User
  name: app-manager
```

---

## Storage

### Volume

- **Volume**은 Kubernetes 스토리지의 **추상화 개념**
- 컨테이너는 **Pod에 바인딩된 Volume을 마운트**하여
    
    → 로컬 파일시스템처럼 스토리지에 접근 가능
    
- Pod 내부 컨테이너 간 데이터 공유 가능

#### 동작 흐름

```
Volume 정의 → Pod에 바인딩 → 컨테이너에서 Mount
```

#### 예시 개념

- HostPath를 Volume으로 선언
- VolumeMount를 통해 컨테이너에서 사용

### HostPath

- **노드(Worker Node)의 파일시스템**을 컨테이너에 마운트
- 주로 **테스트 환경**이나 **싱글 노드 환경**에서 사용
- 멀티 노드 환경에서는 권장되지 않음

#### HostPath type 옵션

| 타입 | 설명 |
| --- | --- |
| DirectoryOrCreate | 경로가 없으면 디렉토리 생성 (권한 0755) |
| Directory | 해당 경로에 디렉토리가 반드시 존재해야 함 |
| FileOrCreate | 경로가 없으면 파일 생성 (권한 0755) |
| File | 해당 경로에 파일이 반드시 존재해야 함 |

### EmptyDir

- emptyDir 볼륨은 **빈 디렉토리**로 시작
- Pod 내부 애플리케이션이 필요한 파일을 생성
- **Pod 삭제 시 데이터도 함께 삭제됨**

#### 특징

- 동일 Pod 내 컨테이너 간 데이터 공유 가능
- 임시 데이터 저장에 적합

#### 사용 사례

- 캐시
- 임시 파일
- 로그 버퍼

### Shared Volume

- 여러 Pod가 동일한 데이터를 참조하는 구조
- Kubernetes 외부의 **공유 스토리지**를 Volume으로 사용

#### 대표 Volume 타입

- AWS EBS
- Azure Disk
- NFS
- 기타 네트워크 스토리지

### NFS (Network File System)

- NFS 서버가 공유하는 디렉토리를
    
    → 여러 Worker Node의 Pod에서 접근 가능
    

#### 구조

```
[NFS Server] → [Worker Node] → [Pod]
```

#### 요구 조건

- NFS 서버에 공유 디렉토리 존재
- Worker Node에 NFS 클라이언트 설치 필요

#### 특징

- 다수 Pod에서 동시 접근 가능
- 상태 저장 애플리케이션에 유용

### PV & PVC

스토리지 관리와 사용을 **역할 분리**하는 구조

#### 개념 구조

```
관리자: PV 생성 (스토리지 준비)
개발자: PVC 요청 (필요 용량 선언)
Pod: PVC를 통해 스토리지 사용
```

#### 구성 요소

| 구성 | 역할 |
| --- | --- |
| PV (PersistentVolume) | 실제 스토리지 리소스 |
| PVC (PersistentVolumeClaim) | 개발자가 요청하는 스토리지 |
| StorageClass | 동적 스토리지 프로비저닝 정의 |

---

## Network

### Docker Container Network

- `docker0`
    - 기본 가상 이더넷 브리지 인터페이스 (172.17.0.0/16 대역)
    - L2 통신 기반 브리지 네트워크
- 컨테이너 생성 시
    - veth(virtual ethernet) 인터페이스 쌍 생성
    - 한쪽은 컨테이너, 한쪽은 호스트의 `docker0` 브리지에 연결
- 모든 컨테이너의 외부 통신은 `docker0`를 통해 수행
- 컨테이너 실행 시
    - 172.17.x.y 형태의 IP 자동 할당

### Container Network Interface (CNI)

- 쿠버네티스에서 컨테이너 네트워크를 구성하기 위한 표준 인터페이스
- 가상 네트워크를 생성하고 네트워크 대역을 분할
- 라우팅 테이블을 통해 멀티 호스트 간 Pod 통신 지원
- 대표적인 CNI 플러그인
    - AWS VPC CNI
    - Azure CNI
    - Calico
    - Cilium
    - Weave Net

### kube-proxy

- Kubernetes Service 네트워크를 처리하는 컴포넌트
- 기본적으로 iptables 기반으로 동작 (환경에 따라 IPVS 모드 가능)
- 노드에서 포트를 리슨하여 외부 통신 가능하게 처리
- Service 생성 시
    - 목적지 Pod로 트래픽을 전달하기 위한
    - iptables 또는 IPVS 규칙을 자동 구성

### CoreDNS

- Kubernetes 클러스터 내부 DNS 서버
- Service 및 Pod 이름 기반 통신 지원
- 기본 kube-dns Service
    - ClusterIP: `10.96.0.10` (환경에 따라 다를 수 있음)
- CoreDNS는 Pod 형태로 실행
- 클러스터 내 모든 Pod가 DNS 사용 가능

#### DNS 네이밍 규칙

- Service 접근

```
service_name.namespace.svc.cluster.local
```

- Pod 접근

```
pod_ip_address.namespace.pod.cluster.local
```

#### Pod DNS 설정

- Pod 단위로 DNS 설정을 커스터마이징 가능

**참고 자료**
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

---

## Logging & Monitoring

### Pod 로그 관리

#### 전통적인 애플리케이션 로그 운영 방식

- 애플리케이션이 **고정된 서버(장비)** 에서 실행됨을 전제로 함
- 로그는 로컬 파일 시스템에 저장
- `logrotate` 같은 시스템 도구를 이용해 로그 관리
    - 일정 기간 저장
    - 파일 용량 기준으로 분할
    - 오래된 로그 삭제

**특징**

- 서버 중심(Server-Centric) 로그 관리
- 노드에 직접 접속해야 로그 확인 가능

#### 쿠버네티스 환경에서의 로그 운영

쿠버네티스는 **Pod가 동적으로 생성/삭제/이동되기 때문에** 다음과 같은 요구사항이 발생

- 애플리케이션이 어느 노드에서 실행되는지 추적 필요
- 자원 사용량(CPU/Memory) 확인 필요
- 응답 속도 및 HTTP 응답 코드 시각화 필요
- 개별 노드 접속 없이 로그 확인 필요

#### kubectl logs

```bash
kubectl logs <pod-name>
kubectl logs-f <pod-name>
```

- 특정 Pod 로그 확인 가능
- `f` 옵션으로 실시간 로그 스트리밍 가능
- 기본적으로 컨테이너 stdout/stderr 로그 확인

**한계점**

- 여러 Pod 로그를 동시에 보기 어려움
- 장기 보관 및 검색 기능 없음

### Stern

#### 개념

- 여러 개의 Pod 로그를 **실시간으로 동시에 모니터링**하는 CLI 도구
- Label 기반으로 로그 필터링 가능

#### 예시

```
stern nginx
stern -l app=nginx
```

#### 장점 및 한계

- 여러 Pod 로그 통합 확인
- 실시간 컬러 출력
- 운영 중 디버깅에 매우 유용
- 중앙 저장/검색 기능은 없음

#### 설치

**참고 자료**
- [https://github.com/stern/stern](https://github.com/stern/stern)

### EFK를 활용한 Kubernetes 로그 관리

Elastic Stack은 **쿠버네티스 클러스터에서 로그를 중앙 집중식으로 수집, 저장, 검색, 시각화하기 위한 도구**

- 클러스터 전체 로그를 자동 수집
- 인덱싱 기반 빠른 검색 지원
- 웹 UI 기반 시각화 및 대시보드 제공
- 장애 분석 및 트러블슈팅에 효과적

**특징**

- 분산 환경에 최적화된 로그 아키텍처
- Pod가 재시작되거나 노드가 변경되어도 로그 추적 가능
- 대규모 트래픽 환경에서 운영 가시성 확보 가능

#### Fluentd

- 각 Node에 **DaemonSet** 형태로 배포
- 모든 Pod 로그 수집
- 로그 정제 및 필터링
- Elasticsearch로 전송

#### Elasticsearch

- 로그 저장소 역할
- 대용량 로그 인덱싱
- 빠른 검색 지원

#### Kibana

- 로그 시각화 도구
- Dashboard 구성 가능
- 검색 및 필터링 UI 제공

#### ELK vs EFK

쿠버네티스 환경에서는 EFK를 더 많이 사용

| 구분 | 구성 요소 | 차이점 |
| --- | --- | --- |
| EFK | Elasticsearch + Fluentd/Fluent Bit + Kibana | 가볍고 Kubernetes 친화적 |
| ELK | Elasticsearch + Logstash + Kibana | 더 강력한 로그 변환 기능 |

#### EFK 구축 실습 참고 문서

- Fluentd DaemonSet 배포
- Elasticsearch 설치
- Kibana 연동
- 인덱스 패턴 생성

**참고 자료**
- [ElasticSearch를 활용한 Kubernetes 로깅 환경 구성](https://waspro.tistory.com/762)

### Kubernetes Dashboard

- **웹 기반 쿠버네티스 사용자 인터페이스 (Web UI)**
- 클러스터 리소스를 브라우저에서 시각적으로 관리 가능
- CLI(`kubectl`) 없이도 리소스 상태 조회 및 일부 작업 수행 가능

#### 주요 기능

- **워크로드 관리**
    - Pod, Deployment, StatefulSet, DaemonSet 등 생성/조회/삭제
- **리소스 모니터링**
    - CPU / Memory 사용량 확인 (Metrics Server 필요)
- **리소스 상세 조회**
    - YAML, 이벤트, 로그 확인
- **네임스페이스 단위 관리**
    - 특정 Namespace 기준으로 리소스 필터링

#### 접근 방식

1. Kubernetes Dashboard 배포
2. ServiceAccount 및 RBAC 설정
3. `kubectl proxy` 또는 NodePort/Ingress를 통해 접속
4. 토큰 기반 로그인 (Bearer Token)

#### 특징

- 학습 및 테스트 환경에서 유용
- 운영 환경에서는 **보안 설정(RBAC, 네트워크 제한)** 필수
- 실무에서는 CLI 및 GitOps 방식이 더 일반적

#### 참고 링크

**참고 자료**
- [Deploy and Access the Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

---

## Autoscaling

Kubernetes는 **리소스 사용량과 워크로드 변화에 따라 자동으로 확장/축소**할 수 있는 다양한 Autoscaling 메커니즘을 제공한다.

Autoscaling은 크게 다음 3가지로 구분한다.

- Cluster Autoscaler (CA)
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)

### Cluster Autoscaler (CA)

Cluster Autoscaler는 **노드(Node) 단위의 확장/축소**를 담당한다.

#### 개념

- Public Cloud (AWS, GCP, Azure) 또는 OpenStack 기반 클러스터에서 주로 사용
- Pod가 리소스를 할당받지 못해 **Pending 상태**가 되면 Worker Node 자동 확장
- Node Pool의 Min / Max 범위 내에서 확장
- 장시간 유휴 상태인 노드는 자동 제거

#### 동작 흐름

1. Pod가 스케줄링 불가 → Pending 발생
2. CA가 감지
3. Node Group(ASG, VMSS 등) 확장
4. Pod 스케줄링 완료

#### 특징

- 인프라 레벨 Autoscaling
- 클라우드 Provider API와 연동 필요
- 실제 VM/EC2 생성이 동반됨 (시간 소요)

### Horizontal Pod Autoscaler (HPA)

HPA는 **Pod Replica 수를 자동 조절**한다.

#### Metrics Server

- Pod / Node의 CPU, Memory 사용량 수집
- Kubernetes API를 통해 Metrics 제공
- 기본 리소스 기반 Autoscaling에 필수

**참고 자료**
- [https://github.com/kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server)

#### 개념

- CPU/Memory 사용률 기반으로 Pod 수 확장
- Deployment / ReplicaSet 대상으로 동작
- 최소/최대 Replica 수는 HPA에서 정의

#### 동작 조건

- 기본 30초 간격으로 메트릭 확인
- 임계값 초과 시 Scale-out
- Scale-out 후 3분 안정화 대기
- Scale-in 후 5분 안정화 대기

**참고 자료**
- [HorizontalPodAutoscaler 연습](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

#### 특징

- 애플리케이션 레벨 Autoscaling
- 무중단 확장 가능
- 가장 많이 사용되는 Autoscaler

### Vertical Pod Autoscaler (VPA)

VPA는 **Pod의 CPU/Memory 리소스를 자동 조정**한다.

#### 개념

- VerticalPodAutoscaler CRD 필요
- Pod 리소스 사용량 분석 후 권장값 산출
- 필요 시 Pod Template 수정

#### 동작 방식

- 약 10초 간격으로 메트릭 수집
- 리소스 부족/과다 사용 감지
- request/limit 값 수정
- Pod 재시작 후 적용

#### 특징

- Pod 개수는 유지
- 리소스 스펙 자동 최적화
- 재시작 발생 (무중단 보장 어려움)

---

## Custom Resource

Kubernetes는 기본 리소스(Pod, Deployment 등) 외에도 사용자가 직접 새로운 리소스를 정의할 수 있도록 확장 기능을 제공한다.

이를 통해 Kubernetes를 **플랫폼처럼 확장**할 수 있다.

### Custom Resource (CR)

Custom Resource는 Kubernetes API를 확장하여 **사용자 정의 리소스 타입을 추가**하는 기능이다.

#### Resource 개념

- Kubernetes에서 Resource는 특정 API 오브젝트 모음을 저장하는 Endpoint
- 예:
    - `/api/v1/pods`
    - `/apis/apps/v1/deployments`
- 각 Resource는 etcd에 상태가 저장됨

#### Custom Resource 개념

- Kubernetes API의 Extension 기능
- 사용자가 Pod, Deployment처럼 새로운 API 리소스를 정의 가능
- 정의 후 kubectl 명령어로 일반 리소스처럼 사용 가능

**예시**

```bash
kubectl get myresources
kubectl describe myresources
```

#### 특징

- Kubernetes API를 수정하지 않고 확장 가능
- 내부적으로 etcd에 저장
- CR을 기반으로 별도의 Controller 구현 가능
- Operator 패턴의 핵심 구성 요소

#### 활용 예시

- 데이터베이스 리소스 정의 (MySQL, Redis 등)
- Kafka Cluster 리소스
- CI/CD 파이프라인 리소스
- 클라우드 인프라 리소스 추상화

#### 참고링크

**참고 자료**
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

### Custom Resource Definition (CRD)

CRD는 Custom Resource를 생성하기 위한 **정의서(스키마 정의)** 이다.

#### 역할

- 새로운 API 그룹/버전/리소스 타입 정의
- OpenAPI 스키마를 통해 구조 검증
- kubectl에서 사용 가능하도록 API 서버에 등록

#### 동작 흐름

1. CRD 생성
2. API Server에 새로운 Resource 등록
3. 사용자가 해당 Custom Resource 생성
4. Controller가 이를 감지하고 로직 수행

#### CRD 구성 요소

- apiVersion
- kind: CustomResourceDefinition
- group
- versions
- scope (Namespaced / Cluster)
- names (plural, singular, kind)

#### 실습 문서

**참고 자료**
- [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

---

## Helm

### Helm이란

- Kubernetes의 **패키지 매니저**
- Kubernetes 애플리케이션을 **Chart 단위로 배포 / 관리**

**참고 자료**
- [Docs Home | Helm](https://helm.sh/docs/)

#### Helm 주요 기능

- 새로운 Chart 생성
- Chart를 패키지(.tgz)로 생성
- Chart Repository와 상호작용
- Kubernetes Cluster에 Chart 설치 / 업그레이드 / 제거
- 애플리케이션 Release Lifecycle 관리

### Helm 설치

#### 설치 가이드

**참고 자료**
- [Installing Helm | Helm](https://helm.sh/ko/docs/intro/install)

#### 설치 방법

**바이너리 설치**

- OS별 Helm 바이너리 다운로드 후 설치

**스크립트 설치**

- Helm 최신 버전을 자동 설치

**패키지 매니저 설치**

| OS | Package Manager |
| --- | --- |
| Mac | brew |
| Windows | chocolatey |
| Linux | apt / dnf / yum / snap |

### Helm 구성 요소

#### Chart

Helm 패키지

Kubernetes 애플리케이션 실행에 필요한 리소스 묶음

예

- Deployment
- Service
- ConfigMap
- Ingress

#### Repository

- Helm Chart 저장소
- Chart를 저장하고 공유
- 대표적인 Repoistory로 Bitnami

### Helm 명령어

#### Repository 관리

| Command | 설명 | Example |
| --- | --- | --- |
| helm repo add | Repository 추가 | helm repo add bitnami [https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami) |
| helm repo list | Repository 목록 확인 | helm repo list |
| helm repo update | Repository 업데이트 | helm repo update |
| helm repo remove | Repository 삭제 | helm repo remove bitnami |

#### Chart 관리

| Command | 설명 | Example |
| --- | --- | --- |
| helm search repo | Repository에서 Chart 검색 | helm search repo nginx |
| helm show chart | Chart 정보 확인 | helm show chart bitnami/nginx |
| helm inspect values | values 확인 | helm inspect values bitnami/nginx |
| helm pull | Chart 다운로드 | helm pull bitnami/nginx |
| helm package | Chart 패키징 (.tgz) | helm package mychart |
| helm create | 새로운 Chart 생성 | helm create mychart |

#### Release 관리

| Command | 설명 | Example |
| --- | --- | --- |
| helm install | Chart 설치 | helm install webserver bitnami/nginx |
| helm list | Release 목록 확인 | helm list |
| helm upgrade | Release 업그레이드 | helm upgrade webserver bitnami/nginx |
| helm uninstall | Release 삭제 | helm uninstall webserver |
| helm rollback | 이전 버전으로 롤백 | helm rollback webserver 1 |
| helm history | Release 변경 이력 | helm history webserver |

### Helm Chart 만들기

#### Chart 생성

- 새 Chart 생성

```bash
helm create [chart-name]
```

- Chart 다운로드

```bash
helm pull [chart]
```

### Chart 구조

```
mychart/
 ├── Chart.yaml
 ├── values.yaml
 ├── charts/
 └── templates/
```

#### Chart.yaml

Chart 메타데이터 정의

#### charts

Dependency Chart 저장

#### templates

Kubernetes 리소스 템플릿

예

- deployment.yaml
- service.yaml
- ingress.yaml
- hpa.yaml

#### values.yaml

Template에 적용되는 변수 정의

### Helm Chart 배포

#### Helm Chart Repository

Chart를 저장하고 공유하는 HTTP 서버

**구성 요소**

```
index.yaml
chart-name.tgz
```

**사용 가능한 저장소**

- GitHub
- AWS S3
- GCS
- HTTP 서버

**동작 방식**

1. Chart 패키징
2. Repository 업로드
3. index.yaml 생성
4. 클라이언트가 다운로드

---
