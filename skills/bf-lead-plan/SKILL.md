---
name: bf-lead-plan
description: Tech Spec을 분석하여 Epic/Story 구조를 생성하고, Story 5개 이상일 때 병렬로 Story 문서를 작성한다. Distribute 조율 패턴.
---

# Lead Plan (Distribute Pattern)

## Overview

승인된 Tech Spec을 분석하여 Epic → Story 구조를 생성하고, 각 Story에 난이도(S/M/L/XL)를 태깅한다. Story 문서가 5개 이상이면 Creator 에이전트를 병렬로 스폰하여 문서를 작성한다.

## When to Use

- `bf-lead-orchestrate`가 스폰
- 직접 호출하지 않는다.

## Prerequisites

- 승인된 Tech Spec: `docs/tech-specs/{TICKET}-tech-spec.md`
- Tech Spec 리뷰 통과 (사람 개입 ① 완료)

## Instructions

### 1. 초기 로딩

- `docs/tech-specs/{TICKET}-tech-spec.md` 읽기
- `docs/conventions.md` 읽기 (있으면)

### 2. Epic 도출

- 기능 단위 또는 도메인 단위로 에픽을 분리한다.
- 에픽 간 의존성을 명시한다.
- 에픽은 순차적으로 실행됨을 기준으로 순서를 결정한다.
- **Story 0개 에픽 처리**: 인프라 준비, 설정 변경 등 Story로 분해할 수 없는 에픽은 sprint-status.yaml에 Story 없이 에픽만 등록하고, `e2e: passed`로 초기화하여 자동 skip 처리한다.

### 3. Story 구조 결정

각 Epic 내 Story를 결정한다:
- Story는 병렬 실행 가능한 단위로 분리한다.
- 각 Story에 명확한 AC를 포함한다.
- Story 간 의존성이 있으면 Dependencies 섹션에 명시한다.

### 4. 난이도 태깅

각 Story에 난이도를 태깅한다:

| 난이도 | 기준 |
|--------|------|
| S (Simple) | 단일 파일, 명확한 AC, 의존성 없음 |
| M (Medium) | 2~3 파일, 모듈 간 연결 있음 |
| L (Large) | 다수 파일, 아키텍처 영향 큼 |
| XL (Complex) | 크로스 레이어, 보안/성능 고려, 설계 판단 포함 |

### 5. Story 문서 작성 (Distribute Pattern)

**Story 5개 이상:**
- Creator 에이전트를 스폰한다 (`model: sonnet`).
- 각 Creator에게 전달:
  - Story AC, Epic 컨텍스트, Story 문서 템플릿 (아래 Output Format 참조)
- Lead가 모든 Creator의 `"done"` + story 파일을 수집한다.

**Story 4개 이하:**
- Lead가 직접 순차적으로 Story 문서를 작성한다.

### 6. Sprint 번호 결정

- `docs/archive/` 디렉토리가 존재하면 기존 스프린트 확인
- 마지막 스프린트 번호 + 1 (예: SPRINT-03 → SPRINT-04)
- 아카이브가 없으면 SPRINT-01로 시작
- 사용자가 Jira 티켓 번호를 제공했으면 해당 번호 사용

### 7. sprint-status.yaml 생성

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
```

> 메트릭 필드 기본값: 모든 필드는 0/null/false로 초기화. 각 필드는 이후 Lead 스킬들이 기록한다:
> - `model_used`, `ralph_retries`, `ralph_approaches`, `ralph_stuck`: bf-lead-implement
> - `review_blockers`, `review_recommended`: bf-lead-review
> - `failure_tag`, `is_regression`, `parent_story`: E2E agent

### 8. 파일 저장 및 커밋

- Story 파일: `docs/stories/{TICKET}-story-{N}.md`
- sprint-status.yaml: `docs/sprint-status.yaml`
- git commit: `docs({TICKET}): create epics and stories`

### 9. Done 신호

- 스폰한 상위 에이전트에 전달: `"done"` + stories/ 경로 + sprint-status.yaml 경로
- 종료 (컨텍스트 소멸).

## Output Format

### Story 문서 템플릿

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
