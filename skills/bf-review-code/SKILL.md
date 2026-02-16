---
name: bf-review-code
description: OCR과 Convention Guard를 사용하여 구현된 코드를 리뷰한다. 컨벤션 준수, 중복 구현, 보안, 테스트 커버리지를 검토하고 결과를 사람에게 제시한다.
---

# Review Code

## Overview

구현 완료된 Story의 코드를 OCR(Open Code Review) + Convention Guard로 다관점 리뷰한다. 리뷰 결과를 사람(개입 ②)에게 제시하여 승인 여부를 판단하게 한다.

## When to Use

- `/implement-story` 완료 후 자동 실행 (난이도 M 이상)
- 사용자가 `/review-code`를 직접 입력했을 때

## Instructions

1. 직전 커밋의 변경 파일 목록을 가져온다.

2. 다음 관점에서 코드 리뷰를 수행한다:
   - **Convention Guard**: docs/conventions.md 기준 컨벤션 준수 여부
   - **중복 검사**: 기존 코드와 중복 구현 여부
   - **테스트 커버리지**: AC 대비 테스트 누락 여부
   - **보안**: 인젝션, 인증 우회, 데이터 노출
   - **코드 품질**: 명명 규칙, 함수 크기, 복잡도

3. 난이도 L/XL이면 Agent Teams discourse(토론)를 실행한다:
   - 리뷰어 간 의견 충돌 시 토론으로 합의 도출
   - 토론 결과를 요약하여 사용자에게 제시

4. 리뷰 결과를 정리한다:
   - 🔴 블로커 (반드시 수정)
   - 🟡 권장 수정
   - 🟢 확인 완료

5. 사용자 판단을 기다린다 (사람 개입 ②):
   - 승인 → sprint-status.yaml review 상태를 `approved`로 업데이트
   - 수정 요청 → 수정 후 재리뷰

## Output Format

리뷰 결과를 대화로 출력한다. 블로커가 있으면 수정 사항을 구체적으로 제시한다.
