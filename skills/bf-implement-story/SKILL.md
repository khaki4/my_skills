---
name: bf-implement-story
description: Story를 TDD로 구현한다. 난이도에 따라 모델을 배당하고 단위 테스트 작성 → Red → 구현 → Green 순서로 진행한다. 구현 완료마다 커밋한다.
---

# Implement Story

## Overview

개별 Story를 TDD 방식으로 구현한다. 난이도에 따라 실행 방식과 모델 배당이 달라진다.

## When to Use

- `/bf-create-e2e` 완료 후 자동 실행 (에픽 내 스토리 병렬)
- 사용자가 `/bf-implement-story`를 직접 입력했을 때

## Prerequisites

- Story 파일 존재: `docs/stories/{STORY-ID}.md`
- sprint-status.yaml에 해당 Story 정의됨
- E2E 테스트 작성 완료 (해당 에픽의 `e2e: written`)
- 프로젝트에 테스트 프레임워크 설정됨

## Instructions

1. 대상 Story 파일을 읽고 AC를 확인한다.

2. sprint-status.yaml에서 Story의 난이도를 확인하고 실행 방식을 결정한다:

   **난이도 S (Simple):**
   - 메인 세션에서 직접 구현 (Sonnet 단독, ralph-loop)
   - Agent Teams 위임 없음

   **난이도 M (Medium):**
   - 메인 세션에서 직접 구현 (Sonnet, ralph-loop)
   - 구현 완료 후 Opus로 OCR 리뷰 수행

   **난이도 L (Large):**
   - 메인 세션에서 Agent Teams Lead에게 위임 (컨텍스트 보존)
   - Lead 에이전트 생성 시 `model: opus` 지정:
     - Story AC 분석 및 구현 계획 수립
     - Sonnet 모델 구현자 teammate 생성 (Task tool의 `model: sonnet` 파라미터 사용)
     - 구현자가 TDD 수행하도록 지시
     - 구현 완료 후 리뷰 수행
   - 메인 세션은 완료 통보만 수신

   **난이도 XL (Complex):**
   - 메인 세션에서 Agent Teams Lead에게 위임 (컨텍스트 보존)
   - Lead 에이전트 생성 시 `model: opus` 지정:
     - Story AC 분석 및 아키텍처 설계
     - 3+ teammates 생성 시 Task tool의 `model` 파라미터로 모델 지정:
       - 구현자: `model: sonnet`
       - 통합 담당: `model: sonnet`
       - 리뷰어: `model: opus`
     - Agent Teams discourse(토론)로 설계 검증:
       - Lead가 설계안을 teammate들에게 공유
       - teammate끼리 SendMessage로 직접 challenge/agree/link
       - Lead가 합의/미합의를 판정하고 최종 설계 확정
     - 각 teammate에게 역할별 작업 할당
     - 통합 및 최종 리뷰 수행
   - 메인 세션은 완료 통보만 수신

3. TDD 사이클을 실행한다 (모든 난이도 공통):
   - 단위 테스트 작성 (AC 기반)
   - **Red 확인**: package.json의 `test` 스크립트 실행 (`npm test` 또는 `npx vitest run` 등)
     - 테스트가 실패하는지 확인
     - 예상대로 실패하지 않으면 테스트 코드 수정
   - 구현
   - **Green 확인**: 동일한 테스트 커맨드 재실행
     - 모든 테스트가 통과하는지 확인
     - 실패하면 구현 수정 후 재실행
   - 리팩토링 (필요 시)
     - 리팩토링 후에도 Green 유지 확인

4. 구현 완료 시:
   - git commit 실행 (Story 단위):
     - 메시지 포맷: `feat: implement {STORY-ID} - {brief description}`
     - 예: `feat: implement PROJ-123-story-1 - add user authentication`
     - Bug fix인 경우: `fix: implement {STORY-ID} - {brief description}`
   - sprint-status.yaml 업데이트 (동시성 주의):
     - **중요**: 업데이트 직전에 파일을 다시 읽어서 최신 상태 확인
     - 다른 Story가 먼저 업데이트했을 수 있음 (병렬 실행)
     - 해당 Story의 tdd 상태만 `done`으로 변경
     - 다른 Story의 상태는 보존
     - read → modify → write 간격을 최소화하여 race condition 위험을 줄인다

5. 난이도 M 이상이면 자동으로 `/bf-review-code`를 실행한다.

6. 난이도 S이면 리뷰 없이 사용자 승인(사람 개입 ②)을 요청한다.

7. 사용자 승인 완료 후:
   - sprint-status.yaml 업데이트 (동시성 주의):
     - 파일을 다시 읽어서 최신 상태 확인
     - 해당 Story의 review 상태를 `approved`로 변경
     - read → modify → write 간격을 최소화한다
   - 같은 에픽 내 모든 Story의 review가 `approved`인지 확인
   - 모든 Story가 승인되었으면 자동으로 `/bf-run-e2e {epic-id}`를 실행한다

## Output Format

- 구현 코드 + 단위 테스트
- git commit
- sprint-status.yaml 업데이트
