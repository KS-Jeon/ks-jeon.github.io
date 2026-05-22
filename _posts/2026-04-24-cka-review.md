---
title: "CKA 취득 후기"
date: 2026-04-24
categories: [DevOps, Kubernetes]
tags: [kubernetes, devops, container, study-notes]
description: "CKA(Certified Kubernetes Administrator) 취득 히스토리"
toc: true
---

## Introduction

CKA는 CNCF가 운영하는 Kubernetes 관리자 자격증이다. 이론 시험이 아닌 실기 기반 시험이라 단순 암기로는 절대 통과할 수 없고, 실제 클러스터를 다루는 손에 박힌 감이 필요하다.

---

## 시험 준비

### 사용한 리소스

| 리소스 | 용도 |
|--------|------|
| TTABAE-LEARN YouTube | 쿠버네티스 기초 개념 정리 (이성미 강사) |
| Udemy - CKA with Practice Tests | 최근 기출 유형 파악 및 심화 개념 정리용 (Mumshad Mannambeth) |
| IT-Kiddie YouTube | 실제 출제 유형과 거의 일치. 특히 신규 도메인 파악에 유효 |
| KodeKloud Ultimate Mock Exam Series | 핵심 모의고사. 총 5회 Mock 풀이 |
| Killercoda | 트러블슈팅 시나리오 반복 훈련 (Node NotReady, kubelet, NetworkPolicy 등) |
| killer.sh | 실전 환경 시뮬레이션 |

### KodeKloud Mock 점수 흐름

- Mock 1: 90+
- Mock 2: 85
- Mock 3: ~75 (컨텍스트 스위칭 실수 多)
- Mock 4: 90+
- Mock 5: ~70 (Helm, Gateway API 등 신규 도메인 약점 노출)

> **참고:** 2025년 2월 이후 CKA 커리큘럼이 업데이트되면서 Helm, Gateway API, CRI/CNI, Kustomize가 추가됐다. Mock 3의 컨텍스트 스위칭 이슈는 실제 시험 방식(SSH 기반 클러스터 접속)과 달라서 크게 신경 쓸 필요 없었다.

### 약점 보완 전략

트러블슈팅(30% 배점)이 CKA의 핵심이자 가장 큰 지뢰밭이다. EKS 같은 Managed K8s만 써온 사람에게는 생소한 영역 — 컨트롤플레인 장애, kubelet 재시작, etcd 복구 같은 로우레벨 시나리오가 주를 이룬다.

```bash
# kubelet 상태 확인 / 재시작 흐름은 손에 박혀야 함
systemctl status kubelet
journalctl -u kubelet -n 50
systemctl restart kubelet
```

---

## 1차 시험: PSI 환경 이슈로 사실상 무효

시험 당일, **PSI 보안 브라우저에서 웹캠 연결이 반복적으로 끊겼다.** 시험관 체크인 과정에서부터 문제가 발생했고, PSI 기술 지원팀 개입으로 시간이 다 날아갔다. 실제로 문제를 풀 수 있는 시간은 120분 중 약 40분 남짓이었고, 10문제 정도밖에 진행하지 못했다.

점수는 당연히 합격선 이하였다.

### 항의 & 재시험 요청

바로 Linux Foundation 고객지원(trainingsupport.linuxfoundation.org)에 영문 티켓을 작성했다.

내용 구성:
- 계정 정보 및 시험 세부사항
- 이슈 타임라인 (웹캠 끊김 시점, PSI 지원팀 개입 시각 등)
- PSI 기술지원 티켓 번호 첨부
- 결과 무효화 + 무상 재응시 요청

이틀 후 LF 측에서 응답이 왔다. 내용은 요약하면:

> "PSI 플랫폼의 네트워크 불안정으로 웹캠/비디오 스트리밍 연결이 끊겼음을 확인함. 예외적으로 재응시 자격을 초기화함. 트레이닝 포털에서 CKA → Resume으로 재예약 가능."

LF 입장에서는 "귀하의 네트워크 문제"라고 표현했지만, 실제 원인은 PSI 서버사이드 또는 보안 브라우저 초기화 이슈였다. (집 데스크탑 + 유선 인터넷 환경이었고, 스펙 문제는 전혀 없었다.) 그냥 책임 회피용 표현으로 이해하고 넘어갔다.

---

## 재시험 준비 & 환경 세팅

재시험 전 환경 세팅을 꼼꼼히 다시 했다.

**체크리스트:**
- PSI 보안 브라우저 삭제 후 재설치
- Windows Defender 실시간 보호 비활성화 (시험 중)
- 블루투스 장치 전부 끄기
- 복사/붙여넣기 동작 사전 검증

### 재시험 직전 공부 플랜

KodeKloud Mock 풀이와 IT-Kiddie 유튜브 강의를 병행하며 출제 유형을 파악하는 데 집중했다.
시험 하루 전에는 마지막 남은 Killer.sh 세션으로 실전 감각을 끌어올렸다.
Mock 5 점수가 만족스럽지 않으면 재예약도 고려했지만, 그대로 밀어붙이기로 했다.

---

## 최종 합격: 75점

**합격.** 66점 합격선 대비 9점 여유.

![CKA Certificate](/assets/img/posts/cka-review/certificate.png)

실기 시험이라 고배점 문제(트러블슈팅, etcd 백업, RBAC 등)에서 부분 점수를 확보하는 게 전략이다. 어렵다 싶으면 빠르게 다음 문제로 넘기고, 풀 수 있는 것부터 확실히 챙기는 방식이 유효하다.

---

## 총정리

### 합격에 실질적으로 도움된 것

1. **IT-Kiddie YouTube** — 실제 출제 유형이랑 패턴이 겹치는 경우가 많음
2. **KodeKloud Mock 반복** — 손 근육에 박힐 때까지
3. **etcd 백업/복구 완벽 숙지** — `--data-dir` + `hostPath` manifest 수정까지 세트로
4. **빠른 포기 판단** — 시간 배분이 곧 점수

### 시험 환경 관련 조언

- PSI 환경 이슈는 생각보다 흔하다. 혼자 당한 게 아님.
- 기술 문제로 시험을 망쳤다면 **즉시 LF 티켓 제출**. PSI 기술지원 티켓 번호도 같이 첨부하면 처리가 빠르다.
- 재응시 자격 복원은 LF 공식 대응 루트가 있음. 포기하지 말고 항의해라.

### 솔직한 소감

시험 내용 자체보다 PSI 환경 이슈로 더 소모된 CKA였다. LF 인증 체계 전반에 대한 신뢰도 확실히 낮아졌고, 이후 CKAD/CKS는 고려하지 않을 생각이다.

그래도 CKA 자체는 의미 있는 자격증이다. 쿠버네티스를 진짜 이해하고 있는지 손으로 증명하는 시험이라는 점에서. 그 과정이 유독 험했기에 더 값지기도 하다.