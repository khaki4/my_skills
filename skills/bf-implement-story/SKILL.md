---
name: bf-implement-story
description: Story를 TDD로 구현한다. 난이도에 따라 모델을 배당하고 단위 테스트 작성 → Red → 구현 → Green 순서로 진행한다. 구현 완료마다 커밋한다.
---

# Implement Story

## Overview

개별 Story를 TDD 방식으로 구현한다. 난이도에 따라 실행 방식과 모델 배당이 달라진다.

## When to Use

- `/create-e2e` 완료 후 자동 실행 (에픽 내 스토리 순차/병렬)
- 사용자가 `/implement-story`를 직접 입력했을 때

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
   - Lead 에이전트 (Opus 모델):
     - Story AC 분석 및 구현 계획 수립
     - Sonnet 모델 구현자 teammate 생성
     - 구현자가 TDD 수행하도록 지시
     - 구현 완료 후 리뷰 수행
   - 메인 세션은 완료 통보만 수신

   **난이도 XL (Complex):**
   - 메인 세션에서 Agent Teams Lead에게 위임 (컨텍스트 보존)
   - Lead 에이전트 (Opus 모델):
     - Story AC 분석 및 아키텍처 설계
     - 3+ teammates 생성 (구현자: Sonnet, 통합 담당: Sonnet, 리뷰어: Opus)
     - Agent Teams discourse(토론)로 설계 검증
     - 각 teammate에게 역할별 작업 할당
     - 통합 및 최종 리뷰 수행
   - 메인 세션은 완료 통보만 수신

3. TDD 사이클을 실행한다 (모든 난이도 공통):
   - 단위 테스트 작성 (AC 기반)
   - Red 확인 (테스트 실패)
   - 구현
   - Green 확인 (테스트 통과)
   - 리팩토링 (필요 시)

4. 구현 완료 시:
   - git commit 실행 (Story 단위)
   - sprint-status.yaml의 해당 Story tdd 상태를 `done`으로 업데이트

5. 난이도 M 이상이면 자동으로 `/review-code`를 실행한다.

6. 난이도 S이면 리뷰 없이 사용자 승인(사람 개입 ②)을 요청한다.

7. 사용자 승인 완료 후:
   - sprint-status.yaml의 해당 Story review 상태를 `approved`로 업데이트
   - 같은 에픽 내 모든 Story의 review가 `approved`인지 확인
   - 모든 Story가 승인되었으면 자동으로 `/bf-run-e2e {epic-id}`를 실행한다

## Output Format

- 구현 코드 + 단위 테스트
- git commit
- sprint-status.yaml 업데이트
