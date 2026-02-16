---
name: bf-create-epics-and-stories
description: 승인된 Tech Spec을 기반으로 Epic, Sprint, Story를 생성한다. 각 Story에 난이도(S/M/L/XL)를 태깅하고 sprint-status.yaml을 생성한다. 에픽은 순차 실행, 스토리는 병렬 실행 구조로 설계한다.
---

# Create Epics and Stories

## Overview

승인된 Tech Spec에서 Epic → Sprint → Story 구조를 생성하고, 각 Story에 난이도를 태깅한다.

## When to Use

- 사용자가 `/bf-create-epics-and-stories`를 입력했을 때
- Tech Spec 리뷰가 승인된 후

## Prerequisites

- 승인된 Tech Spec: `docs/tech-specs/{TICKET}-tech-spec.md`
- Tech Spec 리뷰 통과 (사람 개입 ① 완료)
- `docs/` 디렉토리 존재

## Instructions

1. 메인 세션에서 Agent Teams Lead에게 문서 생성을 위임한다.
   - Task tool로 Lead 생성 시 `model: opus` 지정 (복잡한 분석 작업)
   - 메인 세션은 완료 통보만 수신한다 (컨텍스트 보존).

2. **Agent Teams Lead가 수행하는 작업:**

   a. 승인된 Tech Spec 문서를 읽는다.

   b. Epic을 도출한다:
   - 기능 단위 또는 도메인 단위로 분리
   - 에픽 간 의존성 명시
   - 에픽은 순차적으로 실행됨을 기준으로 순서 결정

   c. 각 Epic 내 Story 구조를 결정한다:
   - Story는 병렬 실행 가능한 단위로 분리
   - 각 Story에 명확한 AC 포함
   - Story 간 의존성이 있으면 명시
   - **Story 0개 에픽 처리**: 인프라 준비, 설정 변경 등 Story로 분해할 수 없는 에픽은 sprint-status.yaml에 Story 없이 에픽만 등록하고, `e2e: passed`로 초기화하여 자동 skip 처리한다. 사용자에게 skip 사유를 안내한다.

   d. 각 Story에 난이도를 태깅한다:
   - **S (Simple)**: 단일 파일, 명확한 AC, 의존성 없음
   - **M (Medium)**: 2~3 파일, 모듈 간 연결 있음
   - **L (Large)**: 다수 파일, 아키텍처 영향 큼
   - **XL (Complex)**: 크로스 레이어, 보안/성능 고려, 설계 판단 포함

   e. 서브에이전트들에게 각 Story 문서 작성을 병렬 위임한다.
   - Story 문서 작성은 단순 작업이므로 `model: sonnet` 사용

   f. Sprint 번호를 결정한다:
   - `docs/archive/` 디렉토리가 존재하면 기존 스프린트 확인
   - 마지막 스프린트 번호 + 1 (예: SPRINT-03 → SPRINT-04)
   - 아카이브가 없으면 SPRINT-01로 시작
   - 또는 사용자가 Jira 티켓 번호를 제공했으면 해당 티켓 번호 사용 (예: PROJ-123)

   g. 결과를 취합하고 sprint-status.yaml을 생성한다:

```yaml
SPRINT-XX:
  epic-1:
    story-1:
      status: todo
      difficulty: S
      tdd: pending
      review: pending
      model_used: null
      ralph_retries: 0
      ralph_approaches: 0
      review_blockers: 0
      review_recommended: 0
      failure_tag: null
      is_regression: false
      parent_story: null
      ralph_stuck: false
    story-2:
      status: todo
      difficulty: M
      tdd: pending
      review: pending
      model_used: null
      ralph_retries: 0
      ralph_approaches: 0
      review_blockers: 0
      review_recommended: 0
      failure_tag: null
      is_regression: false
      parent_story: null
      ralph_stuck: false
    e2e: pending
  epic-2:
    story-3:
      status: todo
      difficulty: L
      tdd: pending
      review: pending
      model_used: null
      ralph_retries: 0
      ralph_approaches: 0
      review_blockers: 0
      review_recommended: 0
      failure_tag: null
      is_regression: false
      parent_story: null
      ralph_stuck: false
    e2e: pending
```

   > 메트릭 필드 기본값 설명:
   > - `model_used`: 실제 사용된 모델 전략 (`"sonnet"` | `"opus-lead"` | `"opus-lead+3"` 등). bf-implement-story가 기록.
   > - `ralph_retries`: Green 검증 실패 재시도 횟수. bf-implement-story가 기록.
   > - `ralph_approaches`: Stuck Detection으로 접근 방식을 전환한 횟수. bf-implement-story가 기록.
   > - `review_blockers`: 🔴 Blocker 건수. bf-review-code가 기록.
   > - `review_recommended`: 🟡 Recommended 건수. bf-review-code가 기록.
   > - `failure_tag`: E2E 실패 태그. bf-run-e2e가 기록 (regression Story만 해당).
   > - `is_regression`: E2E 실패로 자동 생성된 Story 여부. bf-run-e2e가 기록.
   > - `parent_story`: regression일 때 원인 Story ID. bf-run-e2e가 기록.
   > - `ralph_stuck`: Ralph Loop 한도 초과 여부. bf-implement-story가 기록.

   h. Story 파일을 `docs/stories/`에 저장한다.

   i. git commit을 수행한다:
      - 메시지: `docs({TICKET}): create epics and stories`
      - 세션 중단 시 작업 유실 방지를 위해 반드시 커밋한다

3. 메인 세션이 완료 통보를 수신한다.
   - 생성된 에픽/스토리 수와 난이도 분포만 표시
   - 컨텍스트가 깨끗한 상태 유지

4. 사용자에게 에픽/스토리 구조를 확인받는다:
   - 생성된 에픽 목록, 각 에픽 내 스토리 수, 난이도 분포를 표시
   - 사용자가 난이도 태깅 조정이나 에픽 순서 변경을 요청하면 반영
   - 사용자 승인 후 자동으로 첫 번째 에픽의 `/bf-create-e2e`를 실행한다
   - Epic loop가 자동으로 시작됨

## Output Format

- `docs/stories/{TICKET}-story-{N}.md` — 각 Story 파일
- `docs/sprint-status.yaml` — 스프린트 상태 추적

### Story 문서 템플릿

각 Story 파일은 다음 섹션을 포함해야 한다:

```markdown
# {TICKET}-story-{N}: {Story Title}

## Epic
{소속 에픽 ID 및 이름}

## Difficulty
{S | M | L | XL} — {난이도 판정 근거}

## Acceptance Criteria
- [ ] AC 1: {구체적이고 테스트 가능한 기준}
- [ ] AC 2: ...

## Technical Notes
- 변경 대상 파일/모듈
- 의존성 (다른 Story와의 관계)
- 주의사항

## Dependencies
- {의존하는 Story ID} (있는 경우)
```
