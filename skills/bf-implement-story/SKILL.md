---
name: implement-story
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

2. 난이도별 실행 방식을 결정한다:
   - **S**: ralph-loop (Sonnet 단독)
   - **M**: ralph-loop + OCR 리뷰 (Sonnet 구현, Opus 리뷰)
   - **L**: Agent Teams 2 teammates (Opus Lead+리뷰, Sonnet 구현)
   - **XL**: Agent Teams 3 teammates + discourse (Opus Lead+리뷰, Sonnet 구현+통합)

3. TDD 사이클을 실행한다:
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

## Output Format

- 구현 코드 + 단위 테스트
- git commit
- sprint-status.yaml 업데이트
