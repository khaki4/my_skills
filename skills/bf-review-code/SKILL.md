---
name: bf-review-code
description: OCR과 Convention Guard를 사용하여 구현된 코드를 리뷰한다. 컨벤션 준수, 중복 구현, 보안, 테스트 커버리지를 검토하고 결과를 사람에게 제시한다.
---

# Review Code

## Overview

구현 완료된 Story의 코드를 OCR(Open Code Review) + Convention Guard로 다관점 리뷰한다. 리뷰 결과를 사람(개입 ②)에게 제시하여 승인 여부를 판단하게 한다.

## When to Use

- `/bf-implement-story` 완료 후 자동 실행 (난이도 M 이상)
- 사용자가 `/bf-review-code`를 직접 입력했을 때

## Prerequisites

- Story 구현 완료 및 git commit 완료
- sprint-status.yaml에서 해당 Story `tdd: done`
- docs/conventions.md 존재 (없으면 Convention Guard skip하고 기본 리뷰만 수행)

## Instructions

1. 직전 커밋의 변경 파일 목록을 가져온다.

2. 난이도별 리뷰 방식을 결정한다:

   **난이도 M (Medium):**
   - Task tool로 Opus 리뷰어 1명 생성 (`model: opus`)하여 리뷰 위임
   - 메인 세션은 리뷰 결과만 수신한다 (컨텍스트 보존)
   - Convention Guard + 코드 품질 리뷰

   **난이도 L/XL (Large/Complex):**
   - 메인 세션에서 Review Lead에게 위임한다 (컨텍스트 보존):
     - Task tool로 Review Lead 생성 (`model: opus`)
     - 메인 세션은 최종 결과만 수신한다
   - Review Lead가 Agent Teams discourse(토론)를 구성한다:
     - 2~3명의 리뷰어 teammate 생성 (`model: opus`)
     - 리뷰어 역할 예시: Convention Guard, Security Reviewer, Architecture Reviewer
     - 각 리뷰어가 독립적으로 분석 수행
     - 교차 검증 (discourse):
       - Review Lead가 각 리뷰어의 발견사항을 수집하여 공유
       - 리뷰어끼리 SendMessage로 직접 challenge/agree/link
       - Review Lead가 합의/미합의 쟁점을 판정
     - Review Lead가 최종 결과를 메인 세션에 전달

3. 다음 관점에서 코드 리뷰를 수행한다:
   - **Convention Guard**: docs/conventions.md 기준 컨벤션 준수 여부
   - **중복 검사**: 기존 코드와 중복 구현 여부
   - **테스트 커버리지**: AC 대비 테스트 누락 여부
   - **보안**: 인젝션, 인증 우회, 데이터 노출
   - **코드 품질**: 명명 규칙, 함수 크기, 복잡도

4. 리뷰 결과를 정리한다:
   - 🔴 블로커 (반드시 수정)
   - 🟡 권장 수정
   - 🟢 확인 완료

5. 사용자 판단을 기다린다 (사람 개입 ②):
   - 승인 → sprint-status.yaml review 상태를 `approved`로 업데이트
   - 수정 요청 → 수정 후 재리뷰

6. 승인 완료 후, 같은 에픽 내 모든 Story의 review가 `approved`인지 확인한다:
   - 모든 Story가 승인되었으면 자동으로 `/bf-run-e2e {epic-id}`를 실행한다
   - 미승인 Story가 남아있으면 대기한다

## Output Format

리뷰 결과를 대화로 출력한다.

구조:
```
### 🔴 Blocker (N건)
- [파일:라인] 설명 + 수정 방안

### 🟡 Recommended (N건)
- [파일:라인] 설명 + 권장 수정

### 🟢 Confirmed (N건)
- 확인 완료 항목 요약

### 미합의 쟁점 (L/XL만, Agent Teams discourse 결과)
- 쟁점 + 각 리뷰어 의견
```
