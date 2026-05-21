---
title: "Datadog 플랫폼 구조 및 주요 기능 가이드"
date: 2025-07-15 14:30:00 +0900
categories: [Observability, Datadog]
tags: [datadog, observability, apm, logs, metrics]
description: "Datadog 플랫폼의 구조와 주요 기능을 이해하기 위한 간단 안내서"
toc: true
---

## Datadog Introduction
최근 **하이브리드 클라우드 환경이 확산됨에 따라**, 클라우드와 온프레미스를 아우르는 통합 모니터링의 중요성이 커지고 있습니다. **Datadog**은 이러한 흐름에 맞춰 **클라우드, 온프레미스, 하이브리드 환경을 모두 지원하는 SaaS 기반 통합 관측 플랫폼**으로, Agent 설치 만으로 다양한 인프라 전반의 데이터를 손쉽게 수집하고 분석할 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image.png)

## Datadog Regions

Datadog에서 리전의 의미는 사용자의 데이터가 저장되고 처리되는 물리적 위치(데이터 센터 영역)를 의미합니다. 본문에서는 Datadog의 리전 정보와 선택 시 고려 사항을 살펴보도록 합니다.

#### 리전별 도메인 및 위치

각 Region은 독립 인프라 기반으로 운영되며, 리전 간 데이터 공유는 지원되지 않습니다.

| 리전 이름 | 도메인 | 위치 |
| --- | --- | --- |
| US1 | `app.datadoghq.com` | 미국 동부 (버지니아) |
| US3 | `us3.datadoghq.com` | 미국 서부 (오리건) |
| US5 | `us5.datadoghq.com` | 미국 중부 (애리조나) |
| EU1 | `app.datadoghq.eu` | 유럽 (프랑크푸르트) |
| AP1 | `ap1.datadoghq.com` | **일본** |
| AP2 | `ap2.datadoghq.com` | **오스트레일리아** |
| GOV | `app.ddog-gov.com` | 미국 정부용 리전 (FedRAMP High) |

#### 리전 선택 시 고려 사항

- **데이터 주권 및 규제 대응**
    
    리전 선택은 **데이터 주권(Data Sovereignty)**, **규제 준수(Compliance)**, **보안 정책**과 밀접하게 관련되어 있습니다. 각 리전은 **물리적으로 격리된 인프라**에서 운영되므로, 금융·공공·글로벌 기업 환경에서는 리전 위치가 중요한 고려 요소입니다.
    
- **계정 단위 리전 고정**
    
    Datadog 계정은 생성 시 지정한 리전에 **영구적으로 고정**되며, **리전 간 이전이 불가능**합니다. 예를 들어 `ap1.datadoghq.com`에서 생성된 계정은 일본 리전에서만 데이터가 처리됩니다. 이후 리전 변경이 필요한 경우 **새 계정을 별도로 생성해야** 합니다.
    

---

## Datadog Integration - Agent Based

**Datadog Agent**는 사용자의 호스트(서버, VM, 컨테이너 등)에 설치되어 **관측 데이터를 수집하고 Datadog 플랫폼으로 전송**하는 경량 소프트웨어입니다. 다양한 운영체제와 아키텍처를 지원하며, 소스 코드도 공개되어 있어 확장성과 유연성이 뛰어납니다. 

**Datadog Agent는 메트릭(Metric), 로그(Logs), 이벤트(Evnets), 트레이스(Traces)와 같은 다양한 관측 데이터(Observability Data)**를 수집하며 이를 통해 인프라와 애플리케이션의 상태를 실시간으로 모니터링 할 수 있게 해줍니다.

본문에서는 Datadog Agent의 핵심 개념과 구성 방법을 중점적으로 다룹니다.

#### Datadog Agent 구성 요소

**Datadog Agent**는 데이터를 수집하고 Datadog SaaS 플랫폼으로 전송하는 역할을 수행하며, 핵심 구성 요소인 **Collector**와 **Forwarder**, 그리고 기능 확장을 위한 **확장 모듈**로 구성됩니다.

아래는 각 구성 요소와 그 역할에 대한 요약입니다.

| 구성 요소 | 설명 |
| --- | --- |
| **Collector** | Agent의 핵심 모듈로, 다양한 통합(Integrations) 및 체크(Check)를 실행하여 메트릭 데이터를 수집합니다. |
| **Forwarder** | 수집된 데이터를 압축·버퍼링하고, 장애 발생 시 재전송을 처리한 뒤 Datadog 플랫폼으로 안전하게 전달합니다. |
| **DogStatsD** *(선택)* | StatsD 프로토콜을 사용하는 서비스에서 사용자 정의 메트릭을 수집할 수 있도록 하는 UDP 기반 리스너 기능입니다. |
| **APM Agent** *(선택)* | 애플리케이션의 분산 추적 데이터를 수집해 트레이싱 기능을 제공합니다. |
| **Process Agent** *(선택)* | 호스트 내에서 실행 중인 프로세스 정보를 실시간으로 수집 및 분석합니다. |

**StatsD는** 애플리케이션이 실시간으로 생성하는 메트릭(예: 카운트, 타이밍, 게이지 등)을 네트워크를 통해 전송할 수 있도록 돕는 경량 통계 수집 데몬입니다.

#### Datadog Agent의 네트워크 트래픽

Datadog Agent는 다양한 데이터를 수집하고 전송하기 위해 여러 포트를 사용합니다. 이 표는 **인바운드/아웃바운드 구분**, **프로토콜**, **포트별 기능**, 그리고 **실무상 중요성**을 기준으로 정리한 것입니다.

특히 운영 환경에서 필수로 열어야 할 주요 포트는 ⭐ 표시로 강조하였습니다.

| 포트 | 프로토콜 | 방향 | 용도 | 중요성 |
| --- | --- | --- | --- | --- |
| ⭐ **443** | TCP | Outbound | Datadog 백엔드로 메트릭, 로그, APM, 이벤트 전송 | **핵심 통신 채널. 차단 시 에이전트 무용지물** |
| ⭐ **123** | UDP | Outbound | NTP 시간 동기화 (Trace, 로그 타임스탬프 정확도 확보) | **시간 불일치 시 APM/모니터링 오류 발생** |
| ⭐ **8125** | UDP | Inbound | DogStatsD 메트릭 수신 (애플리케이션 → Agent) | **사용자 정의 메트릭 수집 시 필수** |
| ⭐ **8126** | TCP | Inbound | APM Trace 수신기 (애플리케이션 트레이스 수집) | **APM 기능 작동의 핵심 포트** |
| 10516 | TCP | Outbound | TCP 기반 로그 수집 전송 | 로그 수집 시 필요 |
| 10255 | TCP | Outbound | Kubernetes HTTP Kubelet 통신 | 쿠버네티스 메타데이터 수집 시 사용 |
| 10250 | TCP | Outbound | Kubernetes HTTPS Kubelet 통신 | 쿠버네티스 인증 API 호출용 |
| 5000 | TCP | Inbound | `go_expvar` 디버깅용 포트 | 로컬 디버깅 전용 |
| 5001 | TCP | Inbound | IPC API 리스닝 | 내부 에이전트 통신 |
| 5002 | TCP | Inbound | 에이전트 로컬 GUI 포트 | 로컬 UI 접근 시 사용 |
| 5012 | TCP | Inbound | APM 디버깅용 `go_expvar` | APM 상태 모니터링 |
| 6062 | TCP | Inbound | 프로세스 에이전트 디버깅용 | 프로세스 모니터링 상태 확인 |
| 6162 | TCP | Inbound | 프로세스 에이전트 런타임 설정 | 설정 동적 변경 등 제어 목적 |

#### Datadog Agent 설치 요구 사항

Datadog Agent는 시스템, 애플리케이션, 클라우드 인프라로부터 메트릭, 로그, 트레이스(APM) 등의 데이터를 수집하는 핵심 구성 요소입니다. Agent는 설치된 호스트에서 다양한 통합 서비스와 연동되어 실시간 관측 데이터를 Datadog 플랫폼으로 전송합니다.

Agent 설치를 위해서는 다음과 같은 조건을 갖추어야 합니다.

- **Datadog 계정** ([https://app.datadoghq.com](https://app.datadoghq.com/))
- **API 키** (Agent가 데이터를 Datadog에 전송하는 데 필요)
- 인터넷 연결 환경 (port 443 outbound 필수)

별도의 계정 및 환경 구성 없이도, **Datadog Learning Center에서 제공하는 무료 Lab 환경**을 통해 Agent 설치를 직접 실습 해볼 수 있습니다.

**👉 [Agent on a Host Lab 체험하기](https://learn.datadoghq.com/courses/agent-on-host)**

#### Agent on a Host Lab - Agent 설치

1. **Datadog Lab 환경에 로그인 합니다. Username과 Password, API Key는 별도로 메모 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 1.png)

1. [**Datadog 관리 콘솔](https://app.datadoghq.com/account/login?next=%2F)에서 Username과 Password를 입력하여 로그인 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 2.png)

1. **Ctrl + K 단축키로 Install Agents 메뉴를 검색 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 3.png)

1. **Agent 설치 항목에서 Linux를 클릭하여 설치 안내 화면으로 이동 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 4.png)

1. **[Select API Key] 버튼을 클릭 후 Lab에서 제공하는 API Key를 적용 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 5.png)

1. **API Key가 적용된 Install Command를 복사 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 6.png)

1. **Lab 터미널 환경에서 복사한 명령어를 실행 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 7.png)

1. **Agent 설치가 완료 되었습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 8.png)

1. **Datadog Agent의 상태 확인 명령어를 입력합니다.**

```bash
datadog-agent status
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 9.png)

1. **명령어 실행 후 Forwarder > Transaction Successes 항목을 확인하면 Agent가  Datadog API에 데이터를 전송하면서 성공적으로 처리된 요청들이 표시됩니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 10.png)

1. **Forwarder > API Keys status 항목에서는 API Key의 끝자리를 보여줍니다. 내 Datadog 계정에 등록된 API Key와 일치하는지 비교하여 올바른 계정으로 전송되고 있는지 검증할 수 있습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 11.png)

1. 현재 **Agent에 적용 중인 최종 설정 값의 확인**을 위해서는 아래의 명령어를 입력합니다.

```bash
datadog-agent config
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 12.png)

1. **Agent의 env 태그는 DD_ENV 환경 변수에 의해 자동으로 적용**되며 아래의 명령어들로 현재 적용 중인 env 태그와 환경 변수를 알 수 있습니다.

```bash
# 환경 변수 조회 명령어
env |grep DD_ENV

# agent의 env 태그 조회 명령어
datadog-agent config|grep ^env
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 13.png)

#### Agent on a Host Lab - Datadog.yaml 편집

1. Agent는 **Datadog.yaml 편집을 통해 데이터 수집과 관련된 설정을 변경할 수 있습니다**. Lab 화면에서는 **IDE 탭으로 이동**합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 14.png)

1. **Live Processes 기능을 활성화**하기 위해 Datadog.yaml 파일을 살펴볼 것입니다. IDE 화면에서 **Ctrl + F를 눌러 Process Collection을 검색 합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 15.png)

1. **아래의 사진과 같이 yaml 파일을 변경**하여 Live Process 기능을 활성화 합니다. **들여쓰기가 잘못되면 설정이 반영되지 않을 수 있으니 주의합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 16.png)

1. IDE로 설정 변경 후, 터미널에서 **아래의 명령어로 Agent를 재시작 합니다.**

```bash
systemctl restart datadog-agent
```

1. 상태 조회를 위해 **아래의 status 명령어를 입력한 후, Collector 섹션의 설정을 확인**합니다. **Number of processes** 값은 에이전트가 감지한 **호스트에서 실행 중인 프로세스의 수**를 나타냅니다. 이 값은 정상적으로 동작하고 있다는 의미로 **0보다 충분히 큰 값**이어야 합니다.

```bash
datadog-agent status
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 17.png)

1. config 명령어를 통해서도 변경한 Live Process 설정 값을 확인할 수 있습니다.

```bash
datadog-agent config | grep process_collection -A1
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 18.png)

#### Agent on a Host Lab - Log 수집 기능 활성화

1. Log 수집 기능 활성화를 위해 **IDE 화면으로 이동합니다. Ctrl + F 입력 후  Log Collection을 검색합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 19.png)

1. Storedog 서비스는 **대부분 컨테이너 기반 작동되기 때문에 아래와 같이 주석 해제 후 설정을 변경**합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 20.png)

1. **Agent를 재시작합니다.**

```bash
systemctl restart datadog-agent
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 21.png)

1. 상태 조회 명령어 입력 후 Log Agent의 정상 작동 여부를 확인합니다.

```bash
datadog-agent status
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 22.png)

1. **Docker Log 수집을 위해서는 Agent의 사용자(dd-agent)를 docker 그룹에 추가해야 합니다.** 아래의 명령어를 활용하여 그룹 추가 및 Agent를 재시작 합니다.

```bash
# docker 그룹에 dd-agent 사용자 추가
sudo usermod -aG docker dd-agent
# Agent 재시작
systemctl restart datadog-agent
```

1. **상태 조회 명령어를 입력하여 Log Agent의 Docker log 수집 상태를 확인합니다.**

```bash
datadog-agent status
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 23.png)

#### Agent on a Host Lab - APM 수집 기능 활성화

1. APM 수집 기능 활성화를 위해 **IDE 탭으로 이동하여 [찾기] 기능을 통해 apm_config 설정을 검색합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 24.png)

1. **apm_config의 주석을 해제합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 25.png)

1. Storedog은 **로컬 환경이 아닌 Docker 기반 서비스이기 때문에  apm_non_local_traffic의 주석 해제 후 설정을 true로 변경합니다. (로컬 APM의 경우 enabled: true 주석 해제)**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 26.png)

1. **Agent를 재시작 후, 상태 조회 명령어를 입력**하여 APM 수집 상태를 확인합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 27.png)

#### Agent on a Host Lab - Agent Check

Datadog Agent는 **"체크(check)"** 라고 불리는 여러 통합 기능과 함께 설치됩니다.

이 체크들은 Python 스크립트이며, Agent가 이를 실행하여 호스트와 그 위에서 동작하는 서비스로부터 **메트릭, 이벤트, 로그**를 수집합니다.

Agent는 기본적으로 **disk, cpu, memory** 와 같은 몇 가지 핵심 체크를 자동으로 실행하지만 다른 특정 서비스 통합은 약간의 설정이 필요합니다.

**특정 서비스들과 관련된 통합은 `conf.d/` 디렉토리 아래의 설정 conf.yaml을 통해 적용이 가능하며, 이는 Datadog의 방대한 통합 생태계의 일부입니다.**

이번 실습에서는 Agent가 호스트 파일 시스템에서 체크 파일을 어디에 저장하는지 확인하고, **Redis의 체크를 직접 구성해보겠습니다.**

**모든 Agent Check와 관련된 설정 가이드는 아래 링크에서 확인할 수 있습니다.**

**참고 자료**
- [Integrations](https://docs.datadoghq.com/integrations/)

1. IDE 탭으로 이동하여 **`datadog-agent/conf.d/redisdb.d/` 경로로 이동합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 28.png)

1. **conf.yaml.example의 이름을 conf.yaml로 변경합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 29.png)

1. password 설정을 찾아 **주석 해제 후 비밀번호를 입력합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 30.png)

1. 하단으로 이동하여 **`log:` 섹션 전체를 사진과 같이 주석 해제합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 31.png)

1. 터미널로 이동하여 **adm 그룹에 dd-agent 사용자를 추가합니다. 이후 Agent를 재시작합니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 32.png)

1. 상태 조회 명령어를 입력하여 **Redis의 수집 상태를 확인**할 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 33.png)

1. **Logs Agent**에서도 **Redis의 로그 수집 상태를 확인할 수 있습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 34.png)

---

## Datadog Integration - Authentication Based

해당 방식은 **Datadog Agent를 로컬 서버나 워크로드에 설치하지 않고** **클라우드 서비스나 SaaS 플랫폼과의 API 연동을 통해 메트릭, 로그, 이벤트를 수집하는 방식**입니다.

즉, **서버 접근 권한이 없거나 서버가 존재하지 않는 환경에서도 관측 데이터를 확보할 수 있도록** 설계된 통합 방식입니다.

대표적인 활용 대상은 다음과 같습니다.

- AWS, Azure, GCP와 같은 **클라우드 서비스**
- AWS Lambda, Azure Functions 등 **Serverless 환경**
- Amazon RDS, Cloud SQL 등 **관리형 데이터베이스**
- GitHub, Slack, Okta 등 **SaaS 애플리케이션**

#### Authentication Based Integration 적용 케이스

다음과 같은 환경에서는 Datadog Agent를 직접 설치하기 어렵습니다.

| 상황 | 설명 |
| --- | --- |
| 서버 관리 권한이 없음 | Managed DB, SaaS 서비스 등 OS 접근 불가 |
| 서버 자체가 존재하지 않음 | Lambda 기반 애플리케이션, Event 기반 아키텍처 |
| 보안 또는 접근 제약 | 외부 바이너리 설치 불가 정책 |
| 빠른 통합 필요 | 별도 배포 절차 없이 API 연동만으로 수집 가능 |

따라서 Agentless Integration은 다음과 같은 방식으로 동작합니다.

```mathematica
클라우드/서비스 (AWS, Azure, SaaS)
     ↓ API / 이벤트 / 로그 Export
Datadog Integration Endpoint
     ↓
Datadog Metrics / Logs / Events Pipeline
     ↓
Dashboard / Monitor / APM / Alerting
```

즉, **클라우드 서비스에서 노출하는 API·로그·메트릭을 Datadog이 직접 수집하여 통합 관측 환경을 구성합니다.**

별도의 계정 및 환경 구성 없이도, **Datadog Learning Center에서 제공하는 무료 Lab 환경**을 통해 AWS와의 통합을 직접 실습 해볼 수 있습니다.

👉 [Introduction to Monitoring AWS Lab 체험하기](https://learn.datadoghq.com/courses/introduction-to-monitoring-aws)

#### Introduction to Monitoring AWS Lab - CloudFormation Stack 탐색

1. 이 랩은 **CloudFormation 스택이 자동으로 AWS에 배포**되어 있습니다. 랩 터미널에서 아래 명령어를 실행하면 CloudFormation 스택의 상태를 확인할 수 있습니다.

```bash
aws cloudformation describe-stacks --stack-name TechStoriesStack
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 35.png)

1. 해당 그림은 TechStories AWS 계정의 아키텍처입니다. 모든 랩 인프라는 **us-east-1 리전**에 배포되어 있으며, 아래의 구성을 가지고 있습니다.
- **Web Server EC2 인스턴스**는 Next.js 프론트엔드를 호스팅합니다.
- ECS Fargate에는 세 가지 서비스가 실행되고 있습니다. (**referrals-service**, **quotes-api**, **generate-posts-api)**
    
    특히 referrals-service는 두 개의 **DynamoDB 테이블**을 사용합니다.
    
- **TechStories Postgres DB**는 RDS Postgres 인스턴스로, TechStories의 **주요 데이터베이스**입니다.
- **Keyword Insights 서비스**는 **SQS Queue, Lambda 함수, DynamoDB 테이블**을 사용하는 서버리스 구성입니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 36.png)

이제 랩 인프라 구조를 이해했으므로, AWS 인프라에 **적절한 태깅(Tagging) 전략이 적용되어 있는지 확인**해봅시다.

1. 랩 IDE에서 `techstories-cloudformation/ecs-services.yml` 파일을 엽니다. 이 파일은 앞서 설명한 ECS Fargate 서비스들을 배포하는 역할을 합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 37.png)

1. `ecs-services.yml` 파일의 **102번 라인** 근처로 이동하세요. **108번, 148번, 199번 라인**에서 `Tags` 속성을 확인합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 38.png)

1. 랩 IDE에서 `techstories-cloudformation/lambda-dynamodb.yml` 파일을 열고 **131번 라인**으로 이동합니다. `KeywordInsightsFunction` 리소스는 `keyword-insights-processor`라는 이름의 Lambda 함수를 생성합니다. 해당 리소스에 적용된 Tags를 확인해 봅니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 39.png)

1. `techstories-cloudformation/` 디렉터리의 템플릿을 자유롭게 탐색하며 리소스에 어떤 태그가 붙어 있는지 확인해봅니다. IDE 검색 기능에서 Tags를 검색하면 태그가 어디에 정의되어 있는지 쉽게 찾을 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 40.png)

#### Introduction to Monitoring AWS Lab - AWS 및 Datadog 로그인

1. Lab에서 제공하는 [Open AWS Console] 링크를 클릭하여 AWS 콘솔을 엽니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 41.png)

1. Lab에서 제공하는 계정 및 패스워드를 입력 후 로그인 합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 42.png)

1. Lab에서 제공하는 Datadog 링크와 계정 정보로 Datadog Console에 로그인 합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 43.png)

#### Introduction to Monitoring AWS Lab - AWS Integration

1. 메뉴 바에서 [Integration] 클릭 후, AWS 통합에서 Add AWS Account(s) 버튼을 클릭합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 44.png)

1. 새 Datadog AWS 통합을 구성하기 위한 입력 폼이 열립니다. Datadog은 이 폼에서 입력한 값을 바탕으로 CloudFormation 템플릿을 생성합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 45.png)

1. 아래와 같이 설정을 입력 후, [**Launch CloudFormation Template]** 버튼 클릭합니다.
- **Method:** CloudFormation 선택
- **Select AWS Region:** us-east-1
- **Datadog API Key:** robot-datadog-learning-center-admin이 생성한 키 선택
- **Send AWS Logs to Datadog:** Yes
- **Detect security issues:** No

![image.png](/assets/img/posts/datadog-platform-features-guide/image 46.png)

1. CloudFormation 콘솔이 열리고, Quick Create Stack 폼이 표시됩니다. 방금 입력한 값들이 **템플릿 파라미터에 미리 채워져 있습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 47.png)

1. 쭉 스크롤 하여, **IAM Role Name** 드롭다운에서 **dd-integration-cloudformation-role**을 반드시 선택합니다. (선택하지 않으면 IAM 역할을 수임하지 못해 오류가 발생)

![image.png](/assets/img/posts/datadog-platform-features-guide/image 48.png)

1. 두 가지 IAM 관련 경고에 대한 체크박스를 모두 선택한 후, 스택 생성을 시작합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 49.png)

1. 스택 생성이 완료되면, Datadog으로 돌아옵니다. AWS Integration 화면에서 성공적으로 통합이 된 것을 확인할 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 50.png)

1. Log Collection 탭으로 이동합니다. **Lab에서 제공되는 람다 함수의 ARN을 복사 후 붙여넣으세요.** 그리고 [Add] 버튼을 클릭합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 51.png)

1. 이후 **Lambda Cloudwatch Logs 섹션을 활성화 후 저장**합니다. CloudFormation 스택이 생성 완료될 때까지 대기합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 52.png)

#### Introduction to Monitoring AWS Lab - Metrics 지표 탐색

1. 이제 Datadog에서 AWS 통합 데이터를 확인해볼 시간입니다. **Metrics 메뉴의 Summary 탭으로 이동하여 통합 데이터의 메트릭 정보를  확인할 수 있습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 53.png)

1. [Open in Metrics Explorer] 버튼을 클릭하여  `avg:aws.ecs.cpuutilization` 메트릭 지표를 확인 해보도록 합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 54.png)

1. 또한 메트릭 쿼리 오른쪽의 **</> 아이콘**을 클릭하면 Raw Query를 볼 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 55.png)

1. 쿼리에서 메트릭 이름을 다음과 같이 변경합니다. 그래프를 보면 Lambda가 **처음 실행될 때 평균 duration이 길었다가 이후 짧아지는 패턴**이 보일 것입니다. 이는 **Lambda Cold Start(콜드 스타트)** 현상 때문입니다.

```css
avg:aws.lambda.duration{*}
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 56.png)

#### Introduction to Monitoring AWS Lab - Monitor 생성

1. Datadog에서 **Monitors > New Monitor** 로 이동합니다. Monitor 유형이 **Metric** 으로 선택되어 있는지 확인한 뒤 **Configure Metric Monitor** 버튼을 클릭합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 57.png)

1. 이제 `avg:aws.ecs.cpuutilization` 메트릭을 기반으로 모니터를 생성해보겠습니다. 아래의 조건으로 Monitor를 생성합니다.
- Detection Method

![image.png](/assets/img/posts/datadog-platform-features-guide/image 58.png)

- Define the Metirc

![image.png](/assets/img/posts/datadog-platform-features-guide/image 59.png)

- Set alert condition

![image.png](/assets/img/posts/datadog-platform-features-guide/image 60.png)

- Configure notifications & automations

![image.png](/assets/img/posts/datadog-platform-features-guide/image 61.png)

이 외의 기타 옵션 값은 기본으로 두고 Monitor 알람을 생성합니다.

1. 모니터 생성 후 Status 페이지로 이동합니다. 여기서 상태를 확인할 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 62.png)

#### Introduction to Monitoring AWS Lab - Lambda Log 탐색

1. AWS 콘솔에서 **CloudWatch → Log groups** 로 이동하여 로그 그룹 중에서 **/aws/lambda/keyword-insights-processor** 를 찾습니다. 사진과 같이 구독 필터가 생성되어 있어야 CloudWatch 로그가 Datadog Forwarder로 전달된 뒤 Datadog으로 전송되고 있다는 뜻입니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 63.png)

1. Datadog에서 **Logs** 메뉴로 이동합니다.  표시되는 로그들은 모두 TechStories CloudFormation 스택에서 배포된 **keyword-insights-processor Lambda 함수의 CloudWatch 로그**입니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 64.png)

1. 로그 한 줄을 클릭해보면 해당 로그의 **상세 정보 패널에서 다양한 정보를 확인할 수 있습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 65.png)

#### Introduction to Monitoring AWS Lab - Dashboard 탐색

1. Dashboard > Dashboard List 화면으로 이동합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 66.png)

1. **AWS Overview라는 대시보드** 화면을 탐색 합니다. 이후 다양한 AWS Dashboard들도 탐색 해보도록 합니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 67.png)

#### Introduction to Monitoring AWS Lab - Serverless 탐색

1. Infrastructure > Serverless 화면으로 이동합니다. 해당 화면에서는 다양한 Serverless 리소스들을 탐색할 수 있습니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 68.png)

1. keyword-insights-processor라는 이름의 Lambda 함수를 클릭하여 **상세** **지표를 확인할 수 있습니다.**

![image.png](/assets/img/posts/datadog-platform-features-guide/image 69.png)

1. Datadog Serverless CLI를 사용하면 Lambda 함수에 Datadog 계측을 쉽게 적용할 수 있습니다. Lab 터미널에서 다음 명령을 실행합니다.

```bash
datadog-ci lambda instrument \
-f keyword-insights-processor \
-v 107 -e 77 \
--service keyword-insights \
--env monitoring-aws-lab
```

![image.png](/assets/img/posts/datadog-platform-features-guide/image 70.png)

1. 이제 애플리케이션에 트래픽이 더 들어오면, 이 Lambda 함수에 대해 **로그뿐 아니라 호출마다 APM 트레이스(trace)**도 함께 확인할 수 있게 됩니다.

![image.png](/assets/img/posts/datadog-platform-features-guide/image 71.png)

---

## Datadog - 3 Pillars of Observability

데이터 수집과 인프라 통합이 완료되면, 다음 단계는 수집된 데이터를 기반으로 **시스템을 관찰하고, 성능을 분석하고, 문제를 해결하는 일**입니다. Datadog은 이를 위해 **Metrics, APM(Traces), Logs**라는 세 가지 Pillar를 중심으로 종합적인 Observability 환경을 제공합니다.

이 장에서는 단순히 “3 Pillar가 무엇인가”가 아니라, **Datadog이 제공하는 구체적 기능들이 어떻게 이 Pillar를 구성하고 운영 현장에서 어떤 효과를 발휘하는지**를 상세히 다룹니다.

### Metrics - 시스템 상태와 성능을 한눈에 파악하는 기초 데이터

Metrics는 Observability의 출발점입니다. 메모리 사용률, CPU 부하, API 응답 속도, Lambda Duration 등 거의 모든 지표가 시계열(Time-series) 형태로 수집됩니다.

#### 실시간 시계열 분석

Datadog의 Metrics Explorer와 Dashboards는 **실시간(Time-series)** 데이터를 기반으로 시스템의 현재 상태와 추세를 시각적으로 분석할 수 있습니다. 운영팀은 문제 발생 가능성을 예측하고, 특정 시점에서의 부하 변화를 빠르게 파악할 수 있습니다.

#### 태그 기반 필터링(Tagging)

AWS 통합, Agent, Lambda, ECS 등에서 자동으로 수집된 **env, service, version, app** 태그는 동일한 메트릭이라도 서비스, 환경, 버전 단위로 세분화된 관찰을 가능하게 합니다

예시) 

- service=quotes-api **→** CPU만 확인
- env=production **→** 메모리 변화 추세 분석
- version=1.0.1 **→** 배포 이후 Latency 증가 여부 비교

#### 자동 대시보드(OOTB Dashboards)

Datadog은 AWS, ECS, Lambda, RDS 등 주요 서비스에 대해 **즉시 사용 가능한 모니터링 대시보드**를 제공합니다.

운영자는 별도의 설정 없이도 리소스 상태를 전체적으로 파악할 수 있습니다.

#### 알림(Alerts)과 이상 탐지(Anomaly Detection)

단순 임계값 기반 Alert뿐 아니라, 머신러닝 기반의 이상 탐지(Anomaly), 예측 기반(Forecasting) Alert 설정까지 가능합니다. 이는 단순 모니터링을 넘어 **문제를 사전에 감지하여 예방하는 SRE 운영 방식**을 지원합니다.

### 2. APM & Traces - 요청 흐름을 분석하고 병목을 찾는 핵심 도구

APM(Trace)은 마이크로서비스, 서버리스, 분산 아키텍처 환경에서 필수적인 기능입니다.

Metrics는 “현상”을 알려주지만, Traces는 **문제가 실제로 어디에서 발생했는지**를 알려줍니다.

#### 분산 트레이싱(Distributed Tracing)

각 요청(Request)이 어떤 경로를 통해 처리되었는지, 어떤 서비스, 함수, DB 쿼리를 거쳤는지를 트리 구조로 보여줍니다.

이를 통해 다음과 같은 분석이 가능합니다.

- 어떤 API가 느려졌는가?
- 느려진 원인은 특정 DB 쿼리인가? 특정 Lambda 실행인가?
- 서비스 간 호출 중 어디에서 지연이 발생하는가?

#### 서비스 맵(Service Map)

서비스 간 관계를 자동으로 시각화하는 기능입니다.

API 호출 흐름, 의존성 관계, 에러 발생 서비스 등을 실시간으로 파악할 수 있습니다.

#### Span 기반 성능 분석

각 요청 단계(Span)에 대해 Latency, Error, Resource Usage를 분석합니다.

특히 서버리스 환경에서는 Lambda 콜드스타트 여부까지 파악할 수 있습니다.

#### Trace Search & Analytics

요청 ID, 사용자 ID, 태그, 에러 코드 등 다양한 조건으로 트레이스를 검색하고 분석할 수 있습니다.

운영 중 **“특정 조건에서만 느려지는 문제”를 찾을 때 매우 유용**합니다.

**APM은 결국 “왜 느려졌는가?”, “어디가 병목인가?”에 대한 명확한 답을 제공합니다.**

### 3. Logs - 문제의 근본 원인을 파악하는 결정적 데이터

Logs는 인프라와 애플리케이션의 세부 동작 기록입니다.

Metrics에서 이상 징후를 감지하고, APM에서 지연 구간을 찾은 뒤, **최종적으로 근본 원인을 파악하는 단계에서 Logs는 필수적**입니다.

#### 통합 로그 수집

CloudWatch Logs, Datadog Forwarder, EC2 Agent 등 다양한 수집 경로를 지원하며, 정규화된 형태의 로그 스트림으로 통합됩니다.

#### 자동 태깅 및 컨텍스트 결합

로그마다 서비스, 호스트, AWS 리소스, 트레이스 ID(trace_id), 사용자 정보 등이 자동으로 태깅됩니다.

이를 통해 다음과 같은 분석이 가능합니다.

- 특정 Trace와 연결된 로그만 보기
- service=generate-posts-api 로그만 필터링
- error:true 로그만 모아보기

#### Log Patterns & Facets

Datadog은 반복 패턴을 자동으로 분석하고 정형화하여 불필요한 중복 로그를 줄이고 핵심 패턴만 추출하도록 돕습니다.

#### Live Tail

로그를 실시간으로 스트리밍 하여 배포 직후 오류 여부를 즉시 검증할 수 있습니다.

#### Logs → Metrics & Logs ↔ Traces 연계

로그 이벤트를 기반으로 커스텀 메트릭을 생성하거나, 특정 Trace와 연결하여 문제의 전체 흐름을 한눈에 볼 수 있습니다.

Logs는 결국 **Root Cause Analysis의 마지막 퍼즐 조각**입니다.

### 결론: 3 Pillar의 통합이 완성하는 End-to-End Observability

Metrics, APM, Logs는 각각 독립된 기능이지만 Datadog에서는 **이 세 기능이 완전히 통합되어 문제 탐지 → 영향 분석 → 원인 파악 → 해결까지 이어지는 End-to-End Observability를 제공합니다.**

Metrics에서 **이상이 감지**되고,

APM에서 **지연 구간과 병목이 식별**되며,

Logs에서 **근본 원인이 드러나고,**

Dashboards와 Monitors를 통해 **즉시 대응 가능**한 운영 환경이 구축됩니다.

이번 장을 통해 Observability의 핵심 요소들이 실제 운영 환경에서 어떻게 활용되는지 살펴보았으며, 앞서 구축한 AWS, Agent, Serverless 기반의 데이터 수집 환경이 이 세 가지의 Pillar를 통해 **실제 운영 인사이트를 제공하는 체계로 연결됨**을 확인했습니다.

---
