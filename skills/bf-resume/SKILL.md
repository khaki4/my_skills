---
name: bf-resume
description: 중단된 BF 워크플로우를 복구한다. sprint-status.yaml에서 마지막 완료 지점을 찾아 다음 단계부터 재개하거나, --from 옵션으로 특정 Story부터 강제 시작한다.
---

# Resume Workflow

## Overview

세션 중단, context limit 도달, 에러 등으로 BF 워크플로우가 중간에 멈춘 경우, sprint-status.yaml의 상태를 분석하여 적절한 지점부터 워크플로우를 재개한다.

## When to Use

- Claude Code 세션이 중단된 후 워크플로우를 이어서 진행할 때
- 사용자가 `/bf-resume`을 입력했을 때
- 사용자가 `/bf-resume --from {STORY-ID}`로 특정 Story부터 재개하고 싶을 때

## Prerequisites

- `docs/sprint-status.yaml` 존재 — 미존재 시 "진행 중인 스프린트가 없습니다. `/bf-create-tech-spec`으로 새 워크플로우를 시작하세요." 안내
- Story 파일들 존재: `docs/stories/` 디렉토리

## Instructions

1. `docs/sprint-status.yaml`을 읽고 현재 스프린트 상태를 분석한다.

2. `--from {STORY-ID}` 옵션이 주어진 경우:
   - 해당 Story를 찾아 `status`를 `todo`로 리셋한다 (tdd, review도 초기화).
   - 메트릭 필드는 보존한다 (이전 시도의 기록).
   - step 5로 이동하여 해당 Story부터 재개한다.

3. 옵션 없이 실행된 경우, 다음 우선순위로 재개 지점을 자동 판별한다:

   **a) `ralph_stuck: true`인 Story가 있는 경우:**
   - 해당 Story ID와 마지막 에러 정보를 사용자에게 표시
   - 사용자에게 방향 제시를 요청하거나, 해당 Story를 skip할지 확인
   - 사용자 결정에 따라 재개

   **b) `status: in_progress`인 Story가 있는 경우:**
   - 해당 Story는 구현이 중단된 상태
   - git status를 확인하여 uncommitted 변경사항이 있는지 확인
   - 변경사항이 있으면: 사용자에게 "이전 진행 중이던 {STORY-ID}의 미커밋 변경사항이 있습니다. 이어서 진행하시겠습니까?" 확인
   - 변경사항이 없으면: 해당 Story의 `status`를 `todo`로 리셋하고 처음부터 재구현
   - step 5로 이동

   **c) 모든 Story가 `done`이지만 E2E가 미완료인 에픽이 있는 경우:**
   - 해당 에픽의 `e2e` 상태 확인:
     - `pending`: `/bf-create-e2e {epic-id}` 실행
     - `written`: 모든 Story의 `review`가 `approved`인지 확인 → `/bf-run-e2e {epic-id}` 실행
   - step 6으로 이동

   **d) E2E까지 완료된 에픽이 있고, 다음 에픽이 남은 경우:**
   - 다음 에픽의 `/bf-create-e2e {next-epic-id}` 실행

   **e) 모든 에픽의 E2E가 `passed`인 경우:**
   - "모든 에픽이 완료되었습니다. `/bf-archive-sprint`를 실행하세요." 안내

   **f) Story가 하나도 없는 경우 (sprint-status.yaml은 있지만 Story 미생성):**
   - "Story가 아직 생성되지 않았습니다. `/bf-create-epics-and-stories`를 실행하세요." 안내

4. 판별된 재개 지점을 사용자에게 표시한다:
   ```
   📍 워크플로우 재개 지점 분석
   - 스프린트: {SPRINT-ID}
   - 현재 상태: {상태 요약}
   - 재개 지점: {스킬명} ({대상 ID})
   - 이유: {판별 근거}

   계속 진행하시겠습니까?
   ```

5. 사용자 확인 후, 판별된 스킬을 실행한다:
   - Story 재개: `/bf-implement-story {STORY-ID}` 실행
   - E2E 작성: `/bf-create-e2e {epic-id}` 실행
   - E2E 실행: `/bf-run-e2e {epic-id}` 실행

6. 재개된 스킬이 완료되면 기존 워크플로우 체이닝이 자동으로 이어진다.

## Output Format

대화로 재개 지점 분석 결과를 출력하고, 사용자 확인 후 해당 스킬을 실행한다. 별도 파일 생성 없음.
