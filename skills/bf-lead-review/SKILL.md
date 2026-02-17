---
name: bf-lead-review
description: Discourse 패턴으로 다관점 리뷰를 수행한다. Tech Spec 리뷰와 에픽 통합 코드 리뷰를 모두 처리하며, 쟁점 해소 프로토콜에 따라 합의/미합의를 분리한다.
---

# Lead Review (Discourse Pattern)

## Overview

Discourse 조율 패턴으로 다관점 리뷰를 수행하는 Lead 스킬이다. 두 가지 모드를 지원한다:

- **tech-spec 모드**: Tech Spec 문서를 리뷰하고 사람 개입 ①을 위한 결과를 생성한다.
- **epic-review 모드**: 에픽 전체 구현을 통합 리뷰하고, 사람 개입 ②를 리뷰어 에이전트와 함께 수행한 뒤 결정을 전달한다.

## When to Use

- **tech-spec 모드**: `/bf-spec`이 Tech Spec 작성 후 자동 스폰
- **epic-review 모드**: `bf-lead-orchestrate`가 E2E 통과 후 자동 스폰
- 직접 호출하지 않는다.

## Prerequisites

- **tech-spec 모드**: `docs/tech-specs/{TICKET}-tech-spec.md` 존재
- **epic-review 모드**: 해당 에픽의 모든 Story `status: done`, E2E `passed`

## Instructions

### 1. 모드 감지

스폰 시 전달받은 파라미터로 모드를 결정한다:
- `mode: "tech-spec"` + tech-spec 경로
- `mode: "epic-review"` + epic ID + tech-spec 경로

### 2. 초기 로딩

| 모드 | 읽는 파일 |
|------|----------|
| tech-spec | tech-spec.md, conventions.md |
| epic-review | tech-spec.md, conventions.md, 에픽 전체 git diff (`git diff` 기준: 에픽 첫 Story 커밋 이전 ~ 마지막 커밋) |

### 3. 모델 선택

| 모드 | 모델 |
|------|------|
| tech-spec | 항상 Opus (프로젝트 전체 의사결정) |
| epic-review | 에픽 내 L/XL Story 포함 시 Opus, S/M만이면 Sonnet |

### 4. 리뷰어 팀 구성

**tech-spec 모드:**
- Lead가 Tech Spec을 읽고 프로젝트 특성에 맞는 **3가지 리뷰 페르소나를 추천**한다.
  - 예: Architecture Reviewer, Security Reviewer, Performance Reviewer
  - 또는: Backend Engineer, Frontend Engineer, QA Engineer
- 추천된 페르소나 3명을 teammate로 생성한다 (`model: opus`).

**epic-review 모드:**
- Lead가 에픽 변경 범위를 분석하여 **2~3명의 리뷰어를 구성**한다:
  - Convention Guard (필수): `docs/conventions.md` 기준 컨벤션 준수 여부
  - 추가 1~2명: 프로젝트 특성에 따라 Architecture, Security, Performance 중 선택
- 모델은 3단계에서 결정한 모델과 동일하게 적용한다.

### 5. 독립 분석

각 리뷰어가 독립적으로 분석을 수행한다:

**tech-spec 모드 관점:**
- 아키텍처 정합성: 기존 패턴과의 일관성, 모듈 경계 위반
- 보안: 인증/인가, 데이터 노출, 입력 검증
- 성능: 쿼리 최적화, 캐싱 전략
- 테스트 가능성: AC가 테스트 가능한 형태인지, 엣지 케이스 누락
- 변경 영향도: 사이드이펙트, 하위 호환성

**epic-review 모드 관점:**
- Convention Guard: `docs/conventions.md` 기준 준수 여부
- 스토리 간 중복/불일치: 중복 유틸 함수, 네이밍 불일치
- 아키텍처 드리프트: 개별로는 맞지만 합치면 드러나는 구조 문제
- 테스트 커버리지: AC 대비 테스트 누락
- 보안: 인젝션, 인증 우회, 데이터 노출

### 6. Discourse (교차 검증) — 쟁점 해소 프로토콜

다음 우선순위로 쟁점을 해소한다:

**① 리뷰어 직접 대화:**
- Lead가 각 리뷰어의 발견사항을 수집하여 공유한다.
- 리뷰어끼리 `SendMessage`로 직접 challenge/agree/보완한다.
- 합의되면 → Lead에 결론만 보고.

**② 미합의 시 Lead 중재:**
- Lead가 프로젝트 방향성(tech-spec, conventions)을 기준으로 판단한다.
- Lead 결정으로 확정.

**③ 그래도 미합의 → 버린다 (기록):**
- 최종 결과에 "미합의 쟁점"으로 포함한다.
- 더 이상 토큰을 쓰지 않는다.

### 7. 리뷰 결과 통합

발견사항을 분류한다:
- 🔴 **Blocker** (반드시 수정)
- 🟡 **Recommended** (권장 수정)
- 🟢 **Confirmed** (확인 완료)
- 미합의 쟁점 (쟁점 + 각 리뷰어 의견 + Lead 판단 근거)

### 8. 모드별 출력 및 후속 처리

#### tech-spec 모드:

1. `docs/reviews/{TICKET}-tech-spec-review.md`에 저장한다.
   - `docs/reviews/` 디렉토리가 없으면 생성
   - 재리뷰 시에는 `## Re-review ({날짜})` 섹션을 append
2. git commit: `docs({TICKET}): review tech spec`
3. 스폰한 상위 에이전트에 전달:
   - `"done"` + review.md 경로
4. 종료 (컨텍스트 소멸).

#### epic-review 모드:

1. `docs/reviews/{EPIC-ID}-review.md`에 저장한다.
2. sprint-status.yaml 업데이트 (`yq -i` 명령어 사용):
   - 에픽 내 각 Story의 `review_blockers`, `review_recommended` 건수 기록
   ```bash
   yq -i '
     .<SPRINT>.<EPIC>.<STORY>.review_blockers = 2 |
     .<SPRINT>.<EPIC>.<STORY>.review_recommended = 5
   ' docs/sprint-status.yaml
   ```
3. git commit: `docs({EPIC-ID}): review epic`

4. **사람 개입 ② — 리뷰어 에이전트와 함께 수행:**
   - 리뷰 결과를 사람에게 제시한다 (리뷰어 에이전트가 살아있는 상태).
   - 사람이 질문하면 리뷰어 에이전트가 직접 답변할 수 있다.
   - 미합의 쟁점에 대해 사람이 최종 판단한다.

5. 사람의 결정에 따라:

   **승인 시:**
   - sprint-status.yaml: 에픽 내 모든 Story `review: approved` (각 Story에 대해):
     ```bash
     yq -i '.<SPRINT>.<EPIC>.<STORY>.review = "approved"' docs/sprint-status.yaml
     ```
   - 스폰한 상위 에이전트에 전달: `"승인"`
   - 종료 (컨텍스트 소멸).

   **수정 지시 시:**
   - review.md에 **수정 지시 섹션**을 추가한다 (원문 그대로 기록):

     ```markdown
     ## 수정 지시 (사람 개입 ② 결과)

     ### {STORY-ID}
     - {구체적 수정 내용}
     - 근거: {리뷰어 지적 또는 사람 판단}

     ### {STORY-ID}
     - {구체적 수정 내용}
     - 근거: {사람 판단}
     ```

   - git commit: `docs({EPIC-ID}): record modification instructions`
   - 스폰한 상위 에이전트에 전달: `"수정 지시"` + review.md 경로 + 대상 Story 목록
   - 종료 (컨텍스트 소멸).

## Output Format

### review.md 구조

```markdown
# {TICKET 또는 EPIC-ID} Review

## 리뷰 개요
- 모드: {tech-spec | epic-review}
- 리뷰어: {페르소나 목록}
- 날짜: {YYYY-MM-DD}

## 🔴 Blocker (N건)
- [{위치}] 설명 + 수정 방안

## 🟡 Recommended (N건)
- [{위치}] 설명 + 권장 수정

## 🟢 Confirmed (N건)
- 확인 완료 항목 요약

## 미합의 쟁점
| 쟁점 | 입장 A | 입장 B | 근거 |
|------|--------|--------|------|
| ... | ... | ... | ... |

## 수정 지시 (사람 개입 ② 결과, epic-review 모드만)
### {STORY-ID}
- {수정 내용} — 근거: {판단 근거}
```

### Fallback

- Agent Teams 생성 실패 시: 메인 세션(또는 스폰한 상위 에이전트)에서 단일 Opus로 리뷰 수행
- Teammate 무응답 (10분 이상): 해당 역할을 제외하고 나머지 결과로 진행, 제외된 역할을 결과에 명시
