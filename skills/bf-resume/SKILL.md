---
name: bf-resume
description: 중단된 BF 워크플로우를 복구한다. sprint-status.yaml에서 마지막 완료 지점을 분석하여 bf-execute와 동일한 에픽 단위 루프로 재개한다.
---

# Resume Workflow

## Overview

세션 중단, context limit 도달, 에러 등으로 BF 워크플로우가 중간에 멈춘 경우, sprint-status.yaml의 상태를 분석하여 적절한 지점부터 워크플로우를 재개한다. bf-execute와 동일한 에픽 단위 루프를 사용하여 사람-시스템 경계를 유지한다.

## When to Use

- Claude Code 세션이 중단된 후 워크플로우를 이어서 진행할 때
- 사용자가 `/bf-resume`을 입력했을 때
- 사용자가 `/bf-resume --from {EPIC-ID}`로 특정 에픽부터 재개하고 싶을 때

## Prerequisites

- `docs/sprint-status.yaml` 존재 — 미존재 시 "진행 중인 스프린트가 없습니다. `/bf-spec`으로 새 워크플로우를 시작하세요." 안내
- `docs/tech-specs/{TICKET}-tech-spec.md` 존재

## Instructions

### 1. 상태 분석

**yq 전제조건 체크** (최초 1회):
```bash
command -v yq >/dev/null 2>&1 || { echo "❌ yq not installed. Install: brew install yq"; exit 1; }
```

`docs/sprint-status.yaml`을 읽고 현재 스프린트 상태를 분석한다.

### 2. --from 옵션 처리

`--from {EPIC-ID}` 옵션이 주어진 경우:
- 해당 에픽 내 `done`이 아닌 Story를 찾아 `status`를 `todo`로 리셋 (tdd, review도 초기화):
  ```bash
  yq -i '
    .<SPRINT>.<EPIC>.<STORY>.status = "todo" |
    .<SPRINT>.<EPIC>.<STORY>.tdd = "pending" |
    .<SPRINT>.<EPIC>.<STORY>.review = "pending"
  ' docs/sprint-status.yaml
  ```
- 에픽의 e2e 상태도 `pending`으로 리셋:
  ```bash
  yq -i '.<SPRINT>.<EPIC>.e2e = "pending"' docs/sprint-status.yaml
  ```
- 메트릭 필드는 보존한다 (이전 시도의 기록).
- 재개 지점을 **해당 에픽**으로 설정한다.

### 3. 자동 재개 지점 판별

옵션 없이 실행된 경우, 다음 우선순위로 재개 지점을 자동 판별한다 (**위에서부터 순서대로 검사하여 첫 번째 매칭 적용**):

**a) Story가 하나도 없는 경우 (sprint-status.yaml은 있지만 Story 미생성):**
- 재개 지점: **Plan 단계** — orchestrate plan 모드부터 시작

**b) `status: in_progress`인 Story가 있는 경우:**
- git status로 uncommitted 변경사항 확인
- 변경사항 있음: 사용자에게 "이전 진행 중이던 {STORY-ID}의 미커밋 변경사항이 있습니다. 이어서 진행하시겠습니까?" 확인
- 변경사항 없음: 해당 Story의 `status`를 `todo`로 리셋 (`yq -i` 사용)
- 재개 지점: **해당 에픽**

**c) 미완료 에픽이 있는 경우 (일부 Story가 `todo`이거나 e2e/review가 미완):**
- 재개 지점: **해당 에픽** (orchestrate epic 모드가 sprint-status.yaml을 읽고 이미 done인 Story를 건너뜀)

**d) 모든 에픽의 모든 Story가 `review: approved`이고 `e2e: passed`인 경우:**
- "모든 에픽이 완료되었습니다. `/bf-archive-sprint`를 실행하세요." 안내
- 종료.

### 4. 재개 지점 제시

```
워크플로우 재개 지점 분석
- 스프린트: {SPRINT-ID}
- 현재 상태: {상태 요약}
- 재개 에픽: {epic-id} (총 N개 중 M번째)
- skipped Story: {있으면 목록 표시}
- 이유: {판별 근거}

계속 진행하시겠습니까?
```

skipped(stuck) Story가 있으면 함께 표시한다. 이전 실행에서 이미 auto-skip 되었으므로, 사람이 수정을 원하면 에픽 결과 확인 시점에서 modification.md로 지시할 수 있다.

### 5. 재개 실행

사용자 확인 후, 재개 지점에 따라:

**Plan 미완:**
- orchestrate (plan 모드) 스폰 (`model: opus`)
- 전달: tech-spec 경로, conventions.md 경로
- 수신 후: sprint-status.yaml의 에픽/스토리 구조를 사람에게 제시
- 이후 에픽 루프 진입 (Step 6)

**에픽 재개:**
- bf-execute와 동일한 에픽 루프 진입 (Step 6)

### 6. 에픽 루프 (bf-execute와 동일)

재개 에픽부터 순서대로 순회한다. 각 에픽에 대해:

#### 6a. orchestrate (epic 모드) 스폰

- Task tool 사용, `model: opus`
- 전달: `mode: "epic"`, `epic_id`, tech-spec 경로, conventions.md 경로
  - 수정 재실행인 경우: `modification_path` 추가 전달
- 수신 대기: `"done"` + sprint-status.yaml 경로 + review.md 경로

#### 6b. 에픽 결과 제시

orchestrate 완료 후 sprint-status.yaml과 review.md를 읽어 사람에게 제시한다:

```
## Epic {EPIC-ID} 완료

### Story 결과
| Story | Status | Difficulty | Retries | Stuck |
|-------|--------|------------|---------|-------|
| story-1 | done | S | 0 | - |
| story-2 | done | M | 2 | - |
| story-3 | skipped (stuck) | L | 5 | stuck.md 참조 |

### E2E: {passed | escalated | max-regression-cycles}
### Integration Review: Blockers {N}건, Recommended {N}건
### 상세: docs/reviews/{EPIC-ID}-review.md

진행하시겠습니까?
1. 다음 에픽으로 진행
2. 수정 후 재실행 (수정 내용 입력)
3. 워크플로우 중단
```

#### 6c. 사람 판단 처리

사람의 선택에 따라:

**1. 다음 에픽으로 진행:**
- 다음 에픽의 6a로 이동한다.

**2. 수정 후 재실행:**
- 사람이 수정 내용을 텍스트로 입력한다.
- `docs/reviews/{EPIC-ID}-modification.md`에 기록한다.
- git commit: `docs({EPIC-ID}): record modification instructions`
- 같은 에픽에 대해 orchestrate를 epic 모드로 다시 스폰한다 (`modification_path` 전달).
- 6b로 돌아가 결과를 다시 제시한다.

**3. 워크플로우 중단:**
- 현재 상태를 안내하고 종료한다.

### 7. 전체 완료

모든 에픽 완료 후, bf-execute와 동일하게 후처리 단계를 안내한다.

```
워크플로우가 완료되었습니다.

다음 단계:
1. /bf-archive-sprint — 스프린트 아카이빙
2. /bf-metrics — 메트릭 분석 (선택)
3. /bf-update-conventions — 컨벤션 업데이트
```

## Output Format

대화로 재개 지점 분석 결과를 출력하고, 사용자 확인 후 에픽 단위 루프를 실행한다. modification.md 외 별도 파일 생성 없음.
