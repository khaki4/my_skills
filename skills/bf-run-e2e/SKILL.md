---
name: bf-run-e2e
description: 에픽 단위로 E2E 테스트를 실행한다. agent-browser로 브라우저 테스트를 수행하고, 실패 시 새 Story를 추가하여 Story 루프로 복귀한다. 통과 시 에픽 완료 처리한다.
---

# Run E2E

## Overview

에픽 내 모든 Story가 승인된 후, 선행 작성된 E2E 테스트를 실행하여 통합 검증을 수행한다.

## When to Use

- 에픽 내 모든 Story가 approved된 후 자동 실행
- 사용자가 `/run-e2e`를 직접 입력했을 때

## Instructions

1. 대상 에픽의 sprint-status.yaml을 확인한다:
   - 모든 Story의 review가 `approved`인지 검증
   - 미승인 Story가 있으면 실행 중단 및 안내

2. `tests/e2e/{epic-name}/` 의 테스트를 실행한다:
   - agent-browser를 사용하여 실제 브라우저 환경에서 실행

3. 결과를 판정한다:
   - **전체 통과**:
     - sprint-status.yaml의 해당 에픽 e2e를 `passed`로 업데이트
     - 에픽 완료 처리
     - sprint-status.yaml을 확인하여 다음 에픽 존재 여부 확인:
       - 다음 에픽이 있으면: 자동으로 다음 에픽의 `/bf-create-e2e {next-epic-id}`를 실행
       - 스프린트 내 모든 에픽 완료 시: `/bf-archive-sprint` 실행 안내 (사용자 수동 트리거)
   - **실패**:
     - 실패 원인을 분석하여 새 Story를 생성
     - 실패 태그 추가 (spec-gap | impl-bug | test-design | convention-violation | integration)
     - 새 Story를 sprint-status.yaml에 추가
     - 자동으로 새 Story의 `/bf-implement-story {new-story-id}`를 실행하여 Story 루프 재개

## Output Format

- 테스트 실행 결과 요약 (대화 출력)
- sprint-status.yaml 업데이트
- 실패 시: 새 Story 파일 생성
