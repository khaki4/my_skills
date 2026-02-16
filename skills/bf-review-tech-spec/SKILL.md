---
name: bf-review-tech-spec
description: Agent Teams 토론 방식으로 Tech Spec을 다관점 리뷰한다. 보안, 성능, 테스트 커버리지, 아키텍처 정합성 관점에서 검토하고 리뷰 결과를 사람에게 제시한다.
---

# Review Tech Spec

## Overview

생성된 Tech Spec을 Agent Teams 토론 방식으로 다관점 리뷰한다. 리뷰 결과를 사람(개입 ①)에게 제시하여 승인 여부를 판단하게 한다.

## When to Use

- `/bf-create-tech-spec` 완료 직후 자동 실행
- 사용자가 `/bf-review-tech-spec`을 직접 입력했을 때

## Prerequisites

- Tech Spec 문서 존재: `docs/tech-specs/{TICKET-NUMBER}-tech-spec.md`
- 문서가 없으면 `/bf-create-tech-spec`을 먼저 실행해야 함

## Instructions

1. 메인 세션에서 Agent Teams를 구성하여 다관점 리뷰를 수행한다 (컨텍스트 보존).

2. **Agent Teams 구성:**
   - Principal Engineer 생성 시 `model: opus` 지정:
     - Tech Spec 읽고 3가지 리뷰 페르소나 추천
   - 추천된 페르소나 3명을 teammate로 생성 (Task tool의 `model: opus` 파라미터 사용):
     - 예: Architecture Reviewer, Security Reviewer, Performance Reviewer
     - 또는: Backend Engineer, Frontend Engineer, QA Engineer (프로젝트 특성에 따라)

3. **각 리뷰어 teammate가 독립적으로 분석:**
   - 최신 Tech Spec 문서를 읽는다
   - 각자의 관점에서 리뷰 수행:
     - **아키텍처 정합성**: 기존 패턴과의 일관성, 모듈 경계 위반 여부
     - **보안**: 인증/인가, 데이터 노출, 입력 검증
     - **성능**: 쿼리 최적화, 캐싱 전략, N+1 문제
     - **테스트 가능성**: AC가 테스트 가능한 형태인지, 엣지 케이스 누락 여부
     - **변경 영향도**: 사이드이펙트, 하위 호환성

4. **Agent Teams Discourse (교차 검증):**
   - 각 리뷰어가 발견사항을 공유
   - 서로의 발견사항에 대해 challenge/agree/link
   - 합의된 사항 vs 미합의 쟁점을 분리

5. **Principal Engineer가 결과를 통합:**
   - 각 관점별로 발견사항을 정리:
     - 🔴 블로커 (반드시 수정)
     - 🟡 권장 수정
     - 🟢 확인 완료
   - 미합의 쟁점은 명시적으로 표시
   - 메인 세션에 통합된 리뷰 결과 전달

6. 메인 세션이 리뷰 결과를 사용자에게 제시한다.

7. 사용자 판단을 기다린다 (사람 개입 ①):
   - 승인 → `/bf-create-epics-and-stories` 실행 안내
   - 수정 요청 → Tech Spec 수정 후 재리뷰

## Output Format

리뷰 결과를 대화로 출력한다. 별도 파일 저장 없음.
