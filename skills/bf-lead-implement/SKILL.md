---
name: bf-lead-implement
description: Monitor 패턴으로 에픽 내 모든 Story의 구현을 조율한다. 모든 난이도의 Story를 agent에게 위임하고, sprint-status.yaml은 Lead만 갱신한다.
---

# Lead Implement (Monitor Pattern)

## Overview

에픽 내 모든 Story의 TDD 구현을 조율하는 Lead 스킬이다. **모든 난이도의 Story를 agent에게 위임**하고 Lead는 코드를 직접 만지지 않는다. sprint-status.yaml은 이 Lead만 갱신한다 (단일 쓰기 지점).

## When to Use

- `bf-lead-orchestrate`가 에픽 루프에서 스폰
- 직접 호출하지 않는다.

## Prerequisites

- Story 파일: `docs/stories/{TICKET}-story-*.md` (현재 에픽 소속)
- `docs/sprint-status.yaml`
- `docs/conventions.md` (있으면)
- 수정 지시가 있는 경우: `docs/reviews/{EPIC-ID}-review.md`의 수정 지시 섹션

## Instructions

### 1. 초기 로딩

**yq 전제조건 체크** (최초 1회):
```bash
command -v yq >/dev/null 2>&1 || { echo "❌ yq not installed. Install: brew install yq"; exit 1; }
```

- 현재 에픽에 속한 모든 Story 문서를 읽는다.
- `docs/conventions.md`를 읽는다.
- 수정 재구현인 경우: `review.md`의 수정 지시 섹션을 읽는다.
- `docs/sprint-status.yaml`을 읽어서 각 Story의 난이도와 상태를 확인한다.
  - `status: done`인 Story는 건너뛴다.
  - `status: todo` 또는 `in_progress`인 Story만 대상으로 한다.

### 2. 모델 선택 (Lead 자신)

- 에픽 내 L/XL Story가 있으면: Opus
- S/M만이면: Sonnet

> 이 결정은 `bf-lead-orchestrate`가 스폰 시 모델을 지정한다.

### 3. Story Agent 스폰

모든 난이도의 Story를 agent에게 위임한다. Lead는 코드를 직접 만지지 않는다.

#### S/M Story → 단독 Agent (Sonnet)

- Story당 1개의 Sonnet agent를 스폰한다.
- Agent에게 전달하는 정보:
  - Story 문서 내용 (AC, Technical Notes)
  - `docs/conventions.md` 경로
  - 수정 지시가 있는 경우: review.md의 해당 Story 수정 지시 원문
  - Ralph Loop 지침 (아래 "Story Agent용 Ralph Loop 지침" 참조)
  - **"sprint-status.yaml을 수정하지 말 것"**
  - **"`"done"` + commit hash 또는 `"stuck"` + stuck.md로 보고할 것"**

<HARD-GATE>
Story agent는 sprint-status.yaml을 절대 읽거나 수정하지 않는다. 모든 상태 업데이트는 Lead가 "done"/"stuck" 신호를 수신한 후 수행한다. "상태만 빨리 기록하겠다"는 단일 쓰기 지점 원칙을 파괴하는 합리화이다.
</HARD-GATE>

#### L Story → Sub-Lead (Opus) + Implementers (Sonnet)

- Story당 1개의 Opus sub-lead를 스폰한다.
- Sub-lead에게 전달하는 정보:
  - Story 문서 내용
  - conventions.md 경로
  - 수정 지시 (있으면)
  - "Sonnet implementer를 스폰하여 구현을 진행하라"
  - "쟁점 해소 프로토콜에 따라 조율하라" (아래 참조)
  - Ralph Loop 지침
  - **"sprint-status.yaml을 수정하지 말 것"**
  - **"`"done"` + commit hash 또는 `"stuck"` + stuck.md로 보고할 것"**

#### XL Story → Sub-Lead (Opus) + 3+ Teammates

- Story당 1개의 Opus sub-lead를 스폰한다.
- Sub-lead에게 전달하는 정보:
  - Story 문서 내용
  - conventions.md 경로
  - 수정 지시 (있으면)
  - "3+ teammates를 스폰하여 구현, 통합, 리뷰 역할을 분담하라"
  - "discourse로 설계 검증 후 구현을 진행하라"
  - "쟁점 해소 프로토콜에 따라 조율하라"
  - Ralph Loop 지침
  - **"sprint-status.yaml을 수정하지 말 것"**
  - **"`"done"` + commit hash 또는 `"stuck"` + stuck.md로 보고할 것"**

### 4. 병렬 실행 규칙

- Story 문서의 Technical Notes에서 변경 대상 파일을 확인한다.
- **파일 겹침 없음 + 의존성 없음**: 병렬 실행
- **파일 겹침 또는 Dependencies 명시**: 순차 실행 (의존 순서대로)

### 5. 모니터링 루프

sprint-status.yaml 갱신은 CLAUDE.md의 **Read-yq-Verify** 프로토콜을 따른다.

각 Story agent로부터 완료 신호를 수신한다:

**"done" + commit hash 수신 시:**
- sprint-status.yaml 업데이트 (Lead가 직접, `yq -i` 명령어 사용):
  ```bash
  yq -i '
    .<SPRINT>.<EPIC>.<STORY>.status = "done" |
    .<SPRINT>.<EPIC>.<STORY>.tdd = "done" |
    .<SPRINT>.<EPIC>.<STORY>.model_used = "sonnet" |
    .<SPRINT>.<EPIC>.<STORY>.ralph_retries = 1 |
    .<SPRINT>.<EPIC>.<STORY>.ralph_approaches = 0 |
    .<SPRINT>.<EPIC>.<STORY>.ralph_stuck = false
  ' docs/sprint-status.yaml
  ```
  - `model_used`: 실제 사용된 모델 전략 (`"sonnet"` / `"opus-lead"` / `"opus-lead+3"`)
  - `ralph_retries`: agent가 보고한 재시도 횟수
  - `ralph_approaches`: agent가 보고한 접근 전환 횟수

**"stuck" + stuck.md 수신 시:**
- sprint-status.yaml 업데이트:
  ```bash
  yq -i '
    .<SPRINT>.<EPIC>.<STORY>.ralph_stuck = true |
    .<SPRINT>.<EPIC>.<STORY>.ralph_retries = 3 |
    .<SPRINT>.<EPIC>.<STORY>.ralph_approaches = 2
  ' docs/sprint-status.yaml
  ```
- stuck.md를 저장한다.
- **다른 Story들은 계속 진행한다** (stuck Story가 있어도 나머지를 중단하지 않음).

모든 Story가 완료(done 또는 stuck)될 때까지 대기한다.

### 6. Done 신호

모든 Story 완료 후 orchestrate에 보고한다:

**stuck Story 없음:**
- `"done"` + sprint-status.yaml 경로

**stuck Story 있음:**
- `"done (stuck: {STORY-ID}, {STORY-ID})"` + sprint-status.yaml 경로 + stuck.md 경로들

종료 (컨텍스트 소멸).

---

## Story Agent용 Ralph Loop 지침

Story agent에게 전달할 TDD 지침이다. Agent는 이 지침을 그대로 따른다.

### TDD 사이클

<HARD-GATE>
구현 코드를 작성하기 전에 반드시 테스트를 먼저 작성하고 Red(실패)를 확인해야 한다. 테스트와 구현을 동시에 작성하는 것은 TDD 위반이다. "간단하니까 같이 작성해도 된다"는 이 게이트를 우회하는 전형적인 합리화이다.
</HARD-GATE>

1. **단위 테스트 작성** (AC 기반)
2. **Red 확인**: 테스트 실행 → 실패 확인
   - 예상대로 실패하지 않으면 테스트 코드 수정
3. **구현**
4. **Green 확인**: 테스트 재실행 → 통과 확인
   - 실패하면 아래 가드레일에 따라 재시도
5. **리팩토링** (필요 시, Green 유지 확인)
6. **git commit**: `feat({STORY-ID}): {brief description}`
   - Bug fix인 경우: `fix({STORY-ID}): {brief description}`
   - **sprint-status.yaml은 커밋에 포함하지 않는다**

### Ralph Loop 가드레일

**a) 최대 재시도 횟수: 5회**
- Green 검증 실패 → 구현 수정 → 재실행을 최대 5회까지 반복한다.
- `retry_count`를 0부터 시작하여 매 실패마다 1 증가.

**b) 반복 실패 감지 (Stuck Detection)**
- 매 실패 시 에러 메시지의 핵심 내용(에러 타입 + 발생 파일)을 기록한다.
- 직전 시도와 동일한 근본 원인의 에러가 연속 2회 발생하면, 접근 방식을 전환한다:
  1. 테스트 코드의 기대값이 잘못되었는지 재검토
  2. AC 해석이 올바른지 Story 파일 재확인
  3. 구현 전략을 근본적으로 변경 (다른 알고리즘/패턴 적용)
- `approaches_count`를 0부터 시작하여 전환 시마다 1 증가.

**c) 한도 초과 시 → "stuck" 보고**
- `retry_count >= 5`에 도달하면 루프를 즉시 중단한다.
- **stuck.md를 작성한다:**

```markdown
# Stuck Report: {STORY-ID}

## 재시도 횟수: {retry_count}/5
## 접근 전환 횟수: {approaches_count}

## 시도한 접근 방식
1. {접근 1} — {에러 요약}
2. {접근 2} — {에러 요약}

## 마지막 에러
{전체 에러 출력}

## 변경된 파일
{변경 파일 목록}
```

- Lead에 보고: `"stuck"` + stuck.md 경로
- `retry_count`, `approaches_count`를 함께 보고한다.

### 쟁점 해소 프로토콜 (L/XL Story의 Sub-Lead/Teammates에 적용)

1. **Teammates 직접 대화**: `SendMessage`로 직접 challenge/agree/보완
   - 합의됨 → Sub-Lead에 결론만 보고
2. **미합의 시 Sub-Lead 중재**: 프로젝트 방향성(conventions.md) 기준으로 판단
   - Sub-Lead 결정으로 확정
3. **그래도 미합의 → 버린다 (기록)**: 더 이상 토큰을 쓰지 않음

### Agent Teams 실패 시 Fallback (L/XL)

- Task tool 에러 (agent 생성 실패): 단독 Sonnet agent로 fallback (S/M과 동일 방식)
- Teammate 10분 이상 무응답: 해당 teammate 제외하고 나머지로 진행
- 프로세스 비정상 종료: 단독 Sonnet agent로 fallback

## Output Format

- sprint-status.yaml 업데이트 (이 Lead가 구현 단계의 유일한 쓰기 지점)
- `"done"` 또는 `"done (stuck)"` 신호를 orchestrate에 전달
