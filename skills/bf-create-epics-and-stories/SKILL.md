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

- 승인된 Tech Spec: `docs/tech-specs/{TICKET-NUMBER}-tech-spec.md`
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
    story-1: { status: todo, difficulty: S, tdd: pending, review: pending }
    story-2: { status: todo, difficulty: M, tdd: pending, review: pending }
    e2e: pending
  epic-2:
    story-3: { status: todo, difficulty: L, tdd: pending, review: pending }
    e2e: pending
```

h. Story 파일을 `docs/stories/`에 저장한다.

3. 메인 세션이 완료 통보를 수신한다.
   - 생성된 에픽/스토리 수와 난이도 분포만 표시
   - 컨텍스트가 깨끗한 상태 유지

4. 자동으로 첫 번째 에픽의 `/bf-create-e2e`를 실행한다.
   - Epic loop가 자동으로 시작됨

## Output Format

- `docs/stories/{TICKET}-story-{N}.md` — 각 Story 파일
- `docs/sprint-status.yaml` — 스프린트 상태 추적
