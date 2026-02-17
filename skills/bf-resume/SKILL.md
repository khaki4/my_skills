---
name: bf-resume
description: 중단된 BF 워크플로우를 복구한다. sprint-status.yaml에서 마지막 완료 지점을 분석하여 bf-lead-orchestrate를 적절한 단계부터 재개한다.
---

# Resume Workflow

## Overview

세션 중단, context limit 도달, 에러 등으로 BF 워크플로우가 중간에 멈춘 경우, sprint-status.yaml의 상태를 분석하여 적절한 지점부터 워크플로우를 재개한다. 새 Lead 계층 구조에 맞게 `bf-lead-orchestrate`를 스폰하여 복구한다.

## When to Use

- Claude Code 세션이 중단된 후 워크플로우를 이어서 진행할 때
- 사용자가 `/bf-resume`을 입력했을 때
- 사용자가 `/bf-resume --from {STORY-ID}`로 특정 Story부터 재개하고 싶을 때

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

`--from {STORY-ID}` 옵션이 주어진 경우:
- 해당 Story를 찾아 `status`를 `todo`로 리셋 (tdd, review도 초기화):
  ```bash
  yq -i '
    .<SPRINT>.<EPIC>.<STORY>.status = "todo" |
    .<SPRINT>.<EPIC>.<STORY>.tdd = "pending" |
    .<SPRINT>.<EPIC>.<STORY>.review = "pending"
  ' docs/sprint-status.yaml
  ```
- 메트릭 필드는 보존한다 (이전 시도의 기록).
- 재개 지점을 **"해당 Story의 에픽, 4a 스토리 구현 단계"**로 설정한다.

### 3. 자동 재개 지점 판별

옵션 없이 실행된 경우, 다음 우선순위로 재개 지점을 자동 판별한다 (**위에서부터 순서대로 검사하여 첫 번째 매칭 적용**):

**a) Story가 하나도 없는 경우 (sprint-status.yaml은 있지만 Story 미생성):**
- 재개 지점: **Plan 단계** — orchestrate가 `bf-lead-plan`부터 시작

**b) `ralph_stuck: true`인 Story가 있는 경우:**
- 해당 Story ID와 stuck.md 정보를 사용자에게 표시
- 사용자에게 방향 제시를 요청 (AC 수정 / 접근 변경 / Story 스킵 / 에픽 재설계)
- 사용자 결정에 따라 sprint-status.yaml 수정 후 재개 지점 설정

**c) `status: in_progress`인 Story가 있는 경우:**
- git status로 uncommitted 변경사항 확인
- 변경사항 있음: 사용자에게 "이전 진행 중이던 {STORY-ID}의 미커밋 변경사항이 있습니다. 이어서 진행하시겠습니까?" 확인
- 변경사항 없음: 해당 Story의 `status`를 `todo`로 리셋 (`yq -i` 사용)
- 재개 지점: **해당 에픽, 4a 스토리 구현 단계**

**d) 에픽 내 모든 Story가 `done`이고 `e2e: pending`인 경우:**
- 재개 지점: **해당 에픽, 4b E2E 단계**

**e) 에픽의 `e2e: passed`이고 `review: pending`인 Story가 있는 경우:**
- 재개 지점: **해당 에픽, 4c 통합 리뷰 단계**

**f) 에픽의 모든 Story가 `review: approved`이고 다음 에픽이 남은 경우:**
- 재개 지점: **다음 에픽, 4a부터**

**g) 모든 에픽의 모든 Story가 `review: approved`이고 `e2e: passed`인 경우:**
- "모든 에픽이 완료되었습니다. `/bf-archive-sprint`를 실행하세요." 안내
- 종료.

### 4. 재개 지점 제시

```
워크플로우 재개 지점 분석
- 스프린트: {SPRINT-ID}
- 현재 상태: {상태 요약}
- 재개 에픽: {epic-id} (총 N개 중 M번째)
- 재개 단계: {plan / 4a 스토리 구현 / 4b E2E / 4c 통합 리뷰}
- 이유: {판별 근거}

계속 진행하시겠습니까?
```

### 5. bf-lead-orchestrate 스폰

사용자 확인 후, `bf-lead-orchestrate`를 스폰한다 (`model: opus`).

전달 정보:
- tech-spec 경로
- conventions.md 경로
- **`resume_from`**: 재개 지점 (에픽 ID + 단계)
- **`skip_plan: true`** (Story가 이미 존재하면)
- 수정이 필요한 Story가 있으면: 해당 Story 목록 + review.md 경로 (있으면)

orchestrate는 `resume_from` 정보를 받으면:
- 완료된 에픽을 건너뛴다
- 지정된 에픽의 지정된 단계부터 시작한다
- 나머지는 정상 에픽 루프와 동일하게 진행한다

### 6. 완료 수신

orchestrate로부터 `"done"` 수신 시, bf-execute와 동일하게 후처리 단계를 안내한다.

## Output Format

대화로 재개 지점 분석 결과를 출력하고, 사용자 확인 후 bf-lead-orchestrate를 스폰한다. 별도 파일 생성 없음.
