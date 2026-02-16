---
name: bf-run-e2e
description: 에픽 단위로 E2E 테스트를 실행한다. agent-browser로 브라우저 테스트를 수행하고, 실패 시 새 Story를 추가하여 Story 루프로 복귀한다. 통과 시 에픽 완료 처리한다.
---

# Run E2E

## Overview

에픽 내 모든 Story가 승인된 후, 선행 작성된 E2E 테스트를 실행하여 통합 검증을 수행한다.

## When to Use

- 에픽 내 모든 Story가 approved된 후 자동 실행
- 사용자가 `/bf-run-e2e`를 직접 입력했을 때

## Prerequisites

- E2E 테스트 스크립트 존재: `tests/e2e/{epic-name}/*.sh`
- 해당 에픽의 모든 Story `review: approved`
- agent-browser CLI 설치됨 (`agent-browser install` 완료)

## Instructions

1. 대상 에픽의 sprint-status.yaml을 확인한다:
   - 모든 Story의 review가 `approved`인지 검증
   - 미승인 Story가 있으면 실행 중단 및 안내

2. 해당 에픽의 **전체** E2E 테스트를 실행한다 (regression 수정 후 재실행 시에도 전체 재실행하여 회귀 방지).

   `tests/e2e/{epic-name}/*.sh` 스크립트를 순차 실행한다:
   - 각 `.sh` 파일을 `zsh`로 실행
   - agent-browser CLI가 실제 브라우저를 조작
   - `--session` 플래그로 세션 격리 (병렬 실행 시)
   - **인프라 오류 vs 테스트 실패 구분**:
     - **인프라 오류** (regression Story 생성하지 않음, 사용자에게 인프라 문제 안내):
       - exit code 127 (`command not found` — agent-browser/curl 미설치)
       - `ECONNREFUSED` (서버 미시작)
       - `browser not started`, `browser crashed` 패턴
       - 스크립트 syntax error
     - **테스트 실패** (regression Story 생성 대상):
       - assertion 실패 (`is visible` 실패, grep 불일치, diff 차이)
       - `wait` 타임아웃 (UI 요소가 나타나지 않음 — 구현 문제)
       - 예상과 다른 HTTP status code

3. 결과를 판정한다:
   - **전체 통과**:
     - sprint-status.yaml의 해당 에픽 e2e를 `passed`로 업데이트 — **CLAUDE.md의 "sprint-status.yaml 업데이트 프로토콜"을 따른다**
     - 에픽 완료 처리
     - sprint-status.yaml을 확인하여 다음 에픽 존재 여부 확인:
       - 다음 에픽이 있으면: 자동으로 다음 에픽의 `/bf-create-e2e {next-epic-id}`를 실행
       - 스프린트 내 모든 에픽 완료 시: `/bf-archive-sprint` 실행 안내 (사용자 수동 트리거)
   - **실패**:
     - **Regression 가드레일**: 무한 루프를 방지하기 위해 다음 규칙을 적용한다:
       - 해당 에픽 내 `is_regression: true`인 Story 수를 센다.
       - **3개 이상**이면 새 Story를 생성하지 않고 즉시 사용자에게 에스컬레이션한다:
         ```
         ⚠️ Regression Loop 한도 초과 — 사람 개입 필요
         - Epic: {epic-id}
         - 누적 regression Story: {count}/3
         - 마지막 실패 E2E: {failed-test-name}
         - 실패 태그 이력: {tag1, tag2, ...}
         ```
       - 또한 `parent_story` 체인 depth가 **2 이상**인 경우에도 에스컬레이션한다 (regression이 또 regression을 만드는 상황).
       - **depth 계산 방법**: 새 regression Story 생성 시, 원인 Story의 `parent_story`를 확인한다. `parent_story`가 `null`이면 depth=0, 그렇지 않으면 parent의 depth+1로 재귀 계산한다. 예: A(원본)→B(regression)→C(regression)에서 C의 depth=2.
     - 가드레일 통과 시, 실패 원인을 분석하여 새 Story를 생성
     - 실패 태그를 판정하여 추가:
       - `spec-gap`: AC/Tech Spec이 이 시나리오를 예상하지 못함 (누락된 요구사항)
       - `impl-bug`: AC는 맞으나 구현에 결함 (로직 오류, 오타)
       - `test-design`: E2E 테스트 자체가 잘못됨 (잘못된 assertion, 타이밍)
       - `convention-violation`: conventions.md 규칙 위반
       - `integration`: 개별 모듈 정상이나 결합 시 실패 (API 계약 불일치)
     - 새 Story의 난이도를 태깅한다:
       - 원래 Story와 동일한 기준(파일 수/복잡도) 적용
       - 경향 참고: `impl-bug`, `test-design`, `convention-violation` → 보통 S~M / `spec-gap`, `integration` → 원래 Story 난이도 참고 (M~L)
     - 새 Story를 sprint-status.yaml에 추가할 때 메트릭 필드도 함께 설정:
       - `failure_tag`: 위에서 판정한 실패 태그 (`spec-gap` | `impl-bug` | `test-design` | `convention-violation` | `integration`)
       - `is_regression: true`
       - `parent_story`: 원인이 된 원본 Story ID
     - 새 Story 파일을 `docs/stories/{TICKET}-story-{N+1}.md`로 생성한다 (기존 Story 번호에서 순번을 이어감)
     - 자동으로 새 Story의 `/bf-implement-story {new-story-id}`를 실행하여 Story 루프 재개

## Output Format

- 테스트 실행 결과 요약 (대화 출력)
- sprint-status.yaml 업데이트
- 실패 시: `docs/stories/{TICKET}-story-{N+1}.md` — 새 regression Story 파일 생성
