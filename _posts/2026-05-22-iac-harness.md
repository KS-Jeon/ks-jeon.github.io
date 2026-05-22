---
title: "로컬 IaC 하네스 구축기"
date: 2026-05-22
categories: [DevOps, IaC]
tags: [iac, claude-code, codex, gemini, terraform, aws, azure, kubernetes, llm, automation]
description: "세 LLM 모델이 역할을 나눠 IaC 산출물을 정의, 생성, 검토하는 로컬 하네스를 만든 과정을 기록"
toc: true
---

## Introduction

올해 초부터 AI 에이전트에게 작업을 맡기는 방식으로 하네스 엔지니어링이라는 개념이 이야기되기 시작했다. 단순히 프롬프트 한 줄로 결과를 뽑는 게 아니라, 역할과 계약을 명시적으로 정의하고 여러 에이전트가 파이프라인처럼 협력하게 만드는 접근이다. 이걸 인프라 작업에 접목해보면 어떨까 싶었다.

클라우드 인프라 작업을 하다 보면 반복되는 패턴이 있다. 요구사항을 정제하고, 그 스펙에 맞춰 구현한 뒤에 보안·규정 준수 관점에서 다시 검토하는 흐름이다. 이 과정은 매번 사람이 직접 하기엔 단순하고, 그렇다고 완전히 자동화하기엔 판단이 필요한 지점이 많다고 생각했다.

하네스 엔지니어링 개념을 여기에 적용하면 딱 맞겠다고 생각했다. 단순히 AI한테 Terraform 코드를 짜달라고 하는 것이 아니라, 각 단계에 가장 적합한 모델을 배치하고 그 모델이 어디까지 쓸 수 있는지 명시적인 계약으로 고정하는 구조를 만들고 싶었다.

그 결과물이 [iac-harness](https://github.com/KS-Jeon/iac-harness)다.

---

## 하네스 구조

### 오케스트레이션과 역할 분담

전체 흐름은 단순하다.

```text
input → spec → output → review → (loop) → human-approve
```

Claude Code가 이 흐름 전체를 오케스트레이트한다. 각 단계는 역할이 다르고, 역할마다 다른 모델을 배치했다.

| Phase | 역할 | 모델 | 실행 |
|-------|------|------|------|
| Specifier | 요구사항 정제 및 spec 작성 | Claude Opus (high thinking) | Claude Code native agent |
| Implementer | spec → IaC 코드 생성 | GPT-5.5 (medium effort) | Codex CLI (codex-plugin-cc) |
| Reviewer | 보안·규정·컨벤션 검토 | Gemini Pro (medium/high thinking) | Gemini CLI (gemini-plugin-cc) |

Reviewer가 미해결 항목을 남기면 루프 상한 안에서 Specifier 또는 Implementer로 되돌아가 재작업한다. 루프 상한 도달 또는 미해결 항목 소진 시 human approval 전에 멈춘다. 운영 실행(apply, destroy, git push, PR 생성 등)은 사람이 직접 한다.

진입점은 단 하나다. Claude Code에서 `/iac-harness-run <project-id>`를 실행하면 된다.

### AGENTS.md와 CLAUDE.md

`AGENTS.md`는 세 행위자 모두에게 적용되는 불변 규칙을 담는다. 금지 사항, phase 쓰기 경계, human approval 전 금지 동작, 프로토콜 키워드(PASS/FAIL/BLOCKED, Specifier/Implementer/Reviewer 등)를 고정한다.

`CLAUDE.md`는 Claude Code 전용 진입점 계약이다. Claude Code가 이 하네스의 유일한 사용자 진입점임과 동시에 오케스트레이터임을 명시하고, 각 역할의 계약 파일 위치를 가리킨다. Codex와 Gemini는 각각의 플러그인을 통해서만 호출된다.

이 두 파일은 모든 행위자가 읽기 전용으로 취급한다.

### conventions

스택별 규칙 모음이다. `aws.md`, `azure.md`, `kubernetes.md`, `terraform.md` 네 파일로 구성된다.

`input/request.md`의 Tech Stack 값이 이 파일명과 정확히 일치해야 Specifier가 해당 컨벤션을 불러온다. 각 파일은 강제 수준 어휘(`필수`, `자동 적용`, `spec 검토 필수`, `요청 시 적용`, `금지`)로 규칙을 분류하고, 보안 가드레일이 최상위 우선순위를 갖는다.

컨벤션 예시(Terraform):
- `terraform apply/destroy`는 금지. 자동화 산출물에 포함하지 않는다.
- secret은 state plaintext 회피 경로(ARN/name 참조)를 우선한다.
- backend/state 보안 체크리스트는 필수 확인 항목이다.

요청이 컨벤션 가드레일과 충돌하면 Specifier가 추측으로 완화하지 않고 `BLOCKED`를 기록한다.

### templates

각 phase 산출물의 구조를 고정하는 템플릿 파일들이다.

```text
templates/
├─ input/
│  ├─ request.template.md
│  └─ refined-request.template.md
├─ spec/
│  ├─ spec.template.md
│  ├─ acceptance-criteria.template.md
│  ├─ constraints.template.yaml
│  └─ module-contract.template.json
├─ output/
│  ├─ implementation-notes.template.md
│  ├─ precheck.template.md
│  └─ validation-plan.template.md
├─ inventory/
│  ├─ terraform-modules.template.md
│  └─ terraform-state.template.md
└─ review/
   ├─ report.template.md
   ├─ fix-request.template.md
   └─ design-review-request.template.md
```

Implementer와 Reviewer가 각 계약에서 이 템플릿을 참조한다. 산출물 구조가 매번 달라지는 문제를 방지하고, Reviewer가 이전 review 결과와 비교하기 쉽게 만든다.

### 저장소 트리

```text
iac-harness/
├─ AGENTS.md                              # 공통 불변 계약
├─ CLAUDE.md                              # Claude Code 진입점 계약
├─ README.md
├─ .claude/
│  ├─ settings.json.example              # Phase별 write 제한 훅 예시
│  ├─ agents/
│  │  └─ iac-harness-specifier.md        # Specifier 계약
│  ├─ codex/
│  │  └─ iac-harness-implementer-contract.md  # Implementer 계약
│  ├─ gemini/
│  │  └─ iac-harness-reviewer-contract.md     # Reviewer 계약
│  └─ commands/
│     └─ iac-harness-run.md              # 오케스트레이션 커맨드
├─ conventions/
│  ├─ README.md
│  ├─ aws.md
│  ├─ azure.md
│  ├─ kubernetes.md
│  └─ terraform.md
├─ scripts/
│  ├─ bootstrap.sh                       # WSL 표준 환경 구성
│  ├─ check-plugin.sh                    # CLI 및 플러그인 상태 확인
│  ├─ pull.sh                            # consumer 저장소에 하네스 설치
│  ├─ setup-stack-tools.sh               # stack 검증 도구 설치
│  └─ validate-report.sh
└─ templates/
   ├─ input/
   ├─ spec/
   ├─ output/
   ├─ inventory/
   └─ review/
```

---

## 트레이드오프

구축 과정에서 결정이 필요했던 지점들이다.

### 단일 진입점 vs. 역할별 직접 호출

초기에는 Claude Code, Codex, Gemini 각각 사용자가 직접 호출하는 구조를 생각했다. 빠르게 특정 phase만 재실행할 수 있다는 장점이 있었다. 하지만 오케스트레이션 로직이 사용자 머릿속에만 존재하고, phase 경계나 루프 조건이 매 실행마다 달라지는 문제가 생겼다.

Claude Code를 단일 진입점으로 고정하고 Codex와 Gemini는 플러그인을 통해서만 호출하도록 바꿨다. 오케스트레이션 일관성을 얻는 대신 특정 phase만 떼어 실행하려면 커맨드를 수정해야 한다.

### 컨벤션 언어를 영어 → 한국어로 전환

초기 컨벤션 파일은 영어로 작성했다. 기술 문서에서 영어가 더 정확하다고 생각했기 때문이다. 그런데 Specifier와 Reviewer가 컨벤션을 해석할 때 한국어 계약 파일과 영어 컨벤션 사이에서 불필요한 번역 레이어가 생겼다. 전체를 한국어로 통일한 뒤 프로토콜 키워드(PASS, FAIL, BLOCKED 등)와 식별자만 영어로 고정했다.

### Reviewer 에스컬레이션 정책

Reviewer가 기본적으로 `medium thinking`으로 실행하고, 판단이 불확실할 때 `escalation: request-gemini-high-rereview` 플래그를 세워 동일 산출물에 대해 `high thinking`으로 1회 재검토하도록 했다. 처음에는 항상 high thinking을 쓰는 것도 고려했지만, 대부분의 review에서 불필요하게 비용이 높아졌다. 에스컬레이션을 Reviewer 스스로 판단하도록 위임한 것이 실용적이었다.

---

## 향후 계획

현재 하네스는 Claude Code를 오케스트레이터로, Codex CLI와 Gemini CLI를 각각 Implementer와 Reviewer로 고정해 사용한다. 하지만 Claude Code, Codex CLI, Gemini CLI 각각의 에이전트 팀이 독립적으로 같은 워크플로우를 수행할 수 있도록 확장하는 것을 고려하고 있다.

구체적으로는 각 플랫폼이 자기 모델만으로 세 역할을 모두 수행하는 네이티브 팀을 구성하는 것이다.

- Claude Code → Claude native Specifier + Claude native Implementer + Claude native Reviewer
- Codex CLI → Codex native Specifier + Codex native Implementer + Codex native Reviewer
- Gemini CLI → Gemini native Specifier + Gemini native Implementer + Gemini native Reviewer

현재의 구조를 만든 이유 중 하나는 자기 평가 관대성을 막기 위해서다. 같은 모델이 코드를 짜고 직접 검토하면 자기 산출물에 후하게 점수를 주는 경향이 있다. 그래서 현재 하네스는 Implementer와 Reviewer를 다른 모델에 맡긴다. 네이티브 팀 실험은 반대로, 한 플랫폼이 처음부터 끝까지 혼자 맡았을 때 이 편향이 얼마나 나오는지 직접 확인해보려는 것이다.
