---
name: bf-lead-orchestrate
description: Sequence 패턴으로 에픽 루프를 조율한다. lead-plan → lead-implement → E2E → lead-review를 순서대로 트리거하고, 분기만 한다. 분석하지 않는다.
---

# Lead Orchestrate (Sequence Pattern)

## Overview

BF 워크플로우의 핵심 조율자이다. Sequence 조율 패턴으로 에픽 루프를 관리한다. 각 Lead를 순서대로 트리거하고, "done" 신호에 따라 분기만 한다. **분석하지 않는다. 시퀀싱만 한다.**

## When to Use

- `bf-execute`가 스폰
- 직접 호출하지 않는다.

## Prerequisites

- 승인된 Tech Spec: `docs/tech-specs/{TICKET}-tech-spec.md`
- `docs/conventions.md` (있으면)

## Instructions

### 1. 초기 로딩

- `docs/tech-specs/{TICKET}-tech-spec.md` 읽기
- `docs/sprint-status.yaml` 읽기 (있으면 — 수정 재구현인 경우)

### 2. Plan 단계 — bf-lead-plan 스폰

- `bf-lead-plan`을 스폰한다 (`model: opus`).
- 전달: tech-spec 경로, conventions.md 경로
- 수신 대기: `"done"` + stories/ 경로 + sprint-status.yaml 경로
- 종료 후: sprint-status.yaml을 읽어 에픽/스토리 구조를 확인한다.

### 3. 사용자 확인

- sprint-status.yaml의 에픽/스토리 구조와 난이도 태그를 사용자에게 제시한다.
- 사용자가 난이도 태그를 조정하면 sprint-status.yaml을 수정한다.
- 사용자 확인 후 에픽 루프를 시작한다.

### 4. 에픽 루프

sprint-status.yaml의 에픽을 순서대로 순회한다. 각 에픽에 대해 4a → 4b → 4c를 순차 실행한다.

> **Story 0개 에픽 처리**: sprint-status.yaml에 Story가 없는 에픽(인프라 준비 등)은 `e2e: passed`로 초기화되어 있으므로 자동 skip한다.

#### 4a. 스토리 구현 — bf-lead-implement 스폰

- **모델 선택**: 에픽 내 L/XL Story 포함 시 `model: opus`, S/M만이면 `model: sonnet`
- 전달 정보:
  - 에픽 ID, Story 문서 경로 목록
  - conventions.md 경로
  - 수정 재구현인 경우: review.md 경로 + 대상 Story 목록
- 수신 대기: `"done"` 또는 `"done (stuck: {STORY-ID}, ...)"` + sprint-status.yaml 경로

**수신 후 분기:**

| 수신 | 다음 행동 |
|------|----------|
| `"done"` (stuck 없음) | 4b로 진행 |
| `"done (stuck: ...)"` | 사람에게 판단 요청 (아래 참조) |

**stuck 처리 — 사람에게 판단 요청:**

stuck.md를 읽어서 사람에게 제시한다:

```
Story {STORY-ID}가 stuck 상태입니다.
- 재시도: {retry_count}/5
- 접근 전환: {approaches_count}회
- stuck.md: {경로}

선택하세요:
1. AC 수정 → 해당 Story 재시도
2. 접근 변경 지시 → Story 문서 수정 후 재시도
3. Story 스킵 → 에픽에서 제외하고 진행
4. 에픽 재설계 → lead-plan으로 돌아감
```

사람의 선택에 따라:
- **AC 수정 / 접근 변경**: 해당 Story의 status를 `in_progress`로 변경 후 4a 재실행:
  ```bash
  yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "in_progress"' docs/sprint-status.yaml
  ```
- **Story 스킵**: 해당 Story를 에픽에서 제외 (`status: skipped`), 4b로 진행
- **에픽 재설계**: lead-plan을 다시 스폰 (2단계로)

#### 4b. E2E 작성 + 실행 — E2E agent 스폰

E2E agent를 1개 스폰한다 (`model: sonnet`).
전달 정보: 에픽 ID, Story 목록, tech-spec 경로, conventions.md 경로.

E2E agent는 아래 **"E2E Agent 지침"**을 따른다.

**수신 후 분기:**

| 수신 | 다음 행동 |
|------|----------|
| `"passed"` + tests/e2e/ | 4c로 진행 |
| `"failed"` + regression story 목록 | 4a로 돌아감 (regression story 포함) |
| `"escalation"` + 사유 | 사람에게 에스컬레이션 |

#### 4c. 에픽 통합 리뷰 — bf-lead-review 스폰

- **모델 선택**: 에픽 내 L/XL Story 포함 시 `model: opus`, S/M만이면 `model: sonnet`
- `mode: "epic-review"` + epic ID + tech-spec 경로
- 수신 대기: `"승인"` 또는 `"수정 지시"` + review.md 경로 + 대상 Story 목록

**수신 후 분기:**

| 수신 | 다음 행동 |
|------|----------|
| `"승인"` | 에픽 완료, 다음 에픽으로 |
| `"수정 지시"` + review.md + Story 목록 | 해당 Story만 `in_progress`로 변경 (`yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "in_progress"' docs/sprint-status.yaml`), 4a로 돌아감 |

### 5. 전체 완료 — Done 신호

모든 에픽 완료 후:
- `"done"` + sprint-status.yaml 경로를 bf-execute에 전달한다.
- 종료 (컨텍스트 소멸).

---

## E2E Agent 지침

E2E agent에게 전달할 인라인 지침이다. Agent는 이 지침을 그대로 따른다.

### 1. 프로젝트 E2E 타입 판별

프로젝트 루트를 분석하여 타입을 결정한다:

- **브라우저 UI 프로젝트**: `package.json`에 React/Vue/Angular/Svelte/Next.js dependency, `public/index.html` 또는 `src/App.*`, `next.config.*`/`vite.config.*`/`angular.json`
  → agent-browser CLI 기반 E2E 작성
- **API-only 프로젝트**: 백엔드 프레임워크만 존재 (Express/Fastify/NestJS/Django/Flask 등)
  → curl/httpie 기반 API E2E 작성
- **CLI 도구 프로젝트**: `package.json`의 `bin` 필드 존재, CLI 엔트리포인트
  → shell script 기반 CLI E2E 작성
- **E2E 불가 프로젝트**: 라이브러리, 유틸리티 패키지, 순수 SDK
  → E2E skip, `"passed"` 즉시 보고

### 2. E2E 시나리오 도출

- 에픽 내 모든 Story의 AC를 E2E 관점으로 시나리오화
- Happy path + 주요 실패 경로 포함
- 사용자 플로우 기준으로 순서 결정

### 3. E2E 스크립트 작성

- `tests/e2e/{epic-name}/` 디렉토리에 저장
- 파일명: `{scenario-name}.sh` (shell 스크립트)
- 모든 E2E 타입은 shell script로 통일하여 `zsh`로 직접 실행

**브라우저 UI — agent-browser 요소 선택 원칙:**
- 사용: semantic locator (`find role button --name "Submit"`, `find label "Email"`), @ref (snapshot 기반)
- 금지: CSS 셀렉터, XPath, DOM 구현 세부사항

### 4. E2E 실행

작성 즉시 전체 E2E 실행:

**인프라 오류 vs 테스트 실패 구분:**
- **인프라 오류** (regression Story 생성 안함, 사람에게 안내):
  - exit code 127 (`command not found`)
  - `ECONNREFUSED` (서버 미시작)
  - `browser not started`, `browser crashed`
  - 스크립트 syntax error
- **테스트 실패** (regression Story 생성 대상):
  - assertion 실패, wait 타임아웃, 예상과 다른 HTTP status code

### 5. 결과 판정

**전체 통과:**
- sprint-status.yaml 업데이트:
  ```bash
  yq -i '.<SPRINT>.<EPIC>.e2e = "passed"' docs/sprint-status.yaml
  ```
- `"passed"` + tests/e2e/ 경로를 Lead에 보고
- git commit: `test({epic-name}): create and pass e2e tests`

**실패:**
- **Regression 가드레일** 확인:
  - 에픽 내 `is_regression: true`인 Story **3개 이상** → `"escalation"` 보고
  - `parent_story` 체인 depth **2 이상** → `"escalation"` 보고
- 가드레일 통과 시:
  - 실패 원인 분석 + failure tag 분류:
    - `spec-gap`: AC/Tech Spec이 시나리오를 예상하지 못함
    - `impl-bug`: AC는 맞으나 구현에 결함
    - `test-design`: E2E 테스트 자체가 잘못됨
    - `convention-violation`: conventions.md 규칙 위반
    - `integration`: 개별 모듈 정상이나 결합 시 실패
  - 새 regression Story 문서 생성: `docs/stories/{TICKET}-story-{N+1}.md`
    - 번호: stories/ 전체에서 가장 큰 번호 + 1
  - sprint-status.yaml에 새 Story 추가:
    ```bash
    yq -i '.<SPRINT>.<EPIC>.<NEW-STORY> = {
      "status":"todo","difficulty":"S","tdd":"pending","review":"pending",
      "model_used":null,"ralph_retries":0,"ralph_approaches":0,
      "review_blockers":0,"review_recommended":0,
      "failure_tag":"impl-bug","is_regression":true,
      "parent_story":"story-1","ralph_stuck":false
    }' docs/sprint-status.yaml
    ```
    - `failure_tag`: 판정한 태그
    - `is_regression: true`
    - `parent_story`: 원인 Story ID
    - 나머지 필드: 기본값
  - 난이도 태깅: `impl-bug`/`test-design`/`convention-violation` → S~M, `spec-gap`/`integration` → 원본 참고 M~L
  - git commit: `test({epic-name}): add regression story for {failure-tag}`
  - `"failed"` + regression story 목록을 Lead에 보고

### 6. sprint-status.yaml 업데이트 프로토콜

E2E agent는 CLAUDE.md의 **Read-yq-Verify** 프로토콜을 따른다:
1. 수정 전에 sprint-status.yaml을 읽어 현재 상태 확인
2. `yq -i` 명령어로 대상 필드만 수정
3. 수정 후 파일을 읽어 변경 확인

## Output Format

- `"done"` + sprint-status.yaml 경로 (전체 에픽 완료)
- 중간 과정은 파일에만 기록, orchestrate 컨텍스트에 남기지 않음
