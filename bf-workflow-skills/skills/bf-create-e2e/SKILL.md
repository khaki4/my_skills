---
name: create-e2e
description: 에픽 단위로 E2E 테스트 케이스를 선행 작성한다. agent-browser 기반으로 접근성 트리의 @ref를 활용한 테스트 코드를 생성한다.
---

# Create E2E

## Overview

에픽의 Story 구현에 앞서, 해당 에픽의 E2E 테스트 케이스를 선행 작성한다. agent-browser를 사용하여 실제 브라우저 환경에서의 통합 검증을 준비한다.

## When to Use

- 에픽 구현 시작 전 자동 실행
- 사용자가 `/create-e2e`를 직접 입력했을 때

## Instructions

1. 대상 에픽의 모든 Story와 AC를 읽는다.

2. E2E 테스트 시나리오를 도출한다:
   - 각 AC를 E2E 관점에서 시나리오화
   - Happy path + 주요 실패 경로 포함
   - 사용자 플로우 기준으로 시나리오 순서 결정

3. agent-browser 기반 테스트 코드를 작성한다:
   - 접근성 트리의 @ref 기반 요소 선택 (CSS 셀렉터 사용 금지)
   - 각 시나리오별 독립 실행 가능한 구조

4. 테스트 파일을 저장한다:
   - `tests/e2e/{epic-name}/` 디렉토리에 저장
   - 파일명: `{scenario-name}.test.ts`

5. sprint-status.yaml의 해당 에픽 e2e 상태를 `written`으로 업데이트한다.

## Output Format

- `tests/e2e/{epic-name}/*.test.ts` — E2E 테스트 파일
- sprint-status.yaml 업데이트
