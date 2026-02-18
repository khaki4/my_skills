---
name: bf-lead-orchestrate
description: 모드 기반 완전 자율 실행기. plan 모드에서 Story 구조를 생성하고, epic 모드에서 에픽 1개를 자율 실행(implement → E2E → review)한다. 사람과 직접 소통하지 않는다.
---

# Lead Orchestrate (Sequence Pattern)

## Overview

BF 워크플로우의 자율 실행기이다. 모드 기반으로 동작하며, 사람과 직접 소통하지 않는다. 각 모드에서 완전 자율 실행 후 결과 파일과 함께 종료한다.

| 모드 | 입력 | 실행 | 출력 |
|------|------|------|------|
| `plan` | tech-spec, conventions | bf-lead-plan 스폰 → Story 구조 생성 | "done" + sprint-status.yaml + stories/ |
| `epic` | epic-id, [modification.md] | implement → E2E → review | "done" + 결과 파일들 |

**핵심 원칙: 사람과 절대 소통하지 않는다.** 모든 판단은 자동 정책에 따르고, 결과는 파일에 기록된다. 사람과의 소통은 bf-execute(메인 세션)가 담당한다.

## When to Use

- `bf-execute`가 스폰
- 직접 호출하지 않는다.

## Prerequisites

- 승인된 Tech Spec: `docs/tech-specs/{TICKET}-tech-spec.md`
- `docs/conventions.md` (있으면)

## Instructions

### 1. 초기 로딩

**yq 전제조건 체크** (최초 1회):
```bash
command -v yq >/dev/null 2>&1 || { echo "❌ yq not installed. Install: brew install yq"; exit 1; }
```

- `docs/tech-specs/{TICKET}-tech-spec.md` 읽기
- `docs/sprint-status.yaml` 읽기 (있으면)

### 2. 모드 감지

스폰 시 전달받은 파라미터로 모드를 결정한다:
- `mode: "plan"` → Plan 단계 실행
- `mode: "epic"` + `epic_id` + (선택) `modification_path` → Epic 단계 실행

---

## Plan 모드

### P1. bf-lead-plan 스폰

- `bf-lead-plan`을 스폰한다 (`model: opus`).
- 전달: tech-spec 경로, conventions.md 경로
- 수신 대기: `"done"` + stories/ 경로 + sprint-status.yaml 경로

### P2. 완료

- sprint-status.yaml을 읽어 에픽/스토리 구조를 확인한다.
- bf-execute에 전달: `"done"` + sprint-status.yaml 경로 + stories/ 경로
- 종료 (컨텍스트 소멸).

---

## Epic 모드

전달받은 `epic_id`의 에픽을 자율 실행한다. 4a → 4b → 4c 순차 진행.

### E0. Modification 처리 (modification.md가 전달된 경우)

- modification.md를 읽어 수정 대상 Story를 파악한다.
- 해당 Story의 status를 `in_progress`로 변경:
  ```bash
  yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "in_progress"' docs/sprint-status.yaml
  ```
- tdd, review 필드를 초기화하고, e2e도 `pending`으로 리셋:
  ```bash
  yq -i '
    .<SPRINT>.<EPIC>.<STORY>.tdd = "pending" |
    .<SPRINT>.<EPIC>.<STORY>.review = "pending"
  ' docs/sprint-status.yaml
  yq -i '.<SPRINT>.<EPIC>.e2e = "pending"' docs/sprint-status.yaml
  ```

### E1. 스토리 구현 (4a) — bf-lead-implement 스폰

- **모델 선택**: 에픽 내 L/XL Story 포함 시 `model: opus`, S/M만이면 `model: sonnet`
- 전달 정보:
  - 에픽 ID, Story 문서 경로 목록 (status가 `todo` 또는 `in_progress`인 Story만)
  - conventions.md 경로
  - modification.md 또는 review.md 경로 (있으면)
- 수신 대기: `"done"` 또는 `"done (stuck: {STORY-ID}, ...)"` + sprint-status.yaml 경로

**수신 후 자동 판단 (stuck):**

| 조건 | 자동 결정 |
|------|----------|
| stuck 없음 | E2로 진행 |
| stuck Story 있음 + 비stuck Story 존재 | stuck Story를 `status: skipped`로 변경, 나머지로 E2 진행 |
| 전 Story stuck | 모두 `status: skipped`로 변경, E2 진행 (E2E는 제한적) |

stuck Story를 skip 처리:
```bash
yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "skipped"' docs/sprint-status.yaml
```

stuck 정보는 sprint-status.yaml(`ralph_stuck: true`) + stuck.md에 이미 기록되어 있다. bf-execute가 에픽 결과로 사람에게 제시한다.

### E2. E2E 작성 + 실행 (4b) — E2E agent 스폰

> **Story 0개 에픽 처리**: sprint-status.yaml에 done인 Story가 없는 에픽(전체 skipped 등)은 `e2e: passed`로 기록하고 E3로 진행한다.

E2E agent를 1개 스폰한다 (`model: sonnet`).
전달 정보: 에픽 ID, Story 목록, tech-spec 경로, conventions.md 경로.

E2E agent는 아래 **"E2E Agent 지침"**을 따른다.

**수신 후 자동 판단 (E2E):**

| 조건 | 자동 결정 |
|------|----------|
| `"passed"` | E3로 진행 |
| `"failed"` + regression story (E2E 사이클 2회 미만) | regression story로 E1 재실행 |
| `"failed"` + regression story (E2E 사이클 2회 도달) | `e2e: max-regression-cycles` 기록, E3 진행 |
| `"escalation"` (가드레일 초과 또는 인프라 오류) | `e2e: escalated` 기록, E3 진행 |

E2E 사이클 카운트: epic 모드 진입 시 0으로 시작, `"failed"` 수신마다 +1.

### E3. 에픽 통합 리뷰 (4c) — bf-lead-review 스폰

- **모델 선택**: 에픽 내 L/XL Story 포함 시 `model: opus`, S/M만이면 `model: sonnet`
- `mode: "epic-review"` + epic ID + tech-spec 경로
- 수신 대기: `"done: approved"` 또는 `"done: blockers"` + review.md 경로

**수신 후 자동 판단 (리뷰):**

| 조건 | 자동 결정 |
|------|----------|
| `"done: approved"` (Blocker 0건) | 에픽 완료 |
| `"done: blockers"` (Blocker 1건+) | sprint-status.yaml + review.md에 기록된 상태 유지, **자동 수정 안 함** |

Blocker가 있어도 자동 수정하지 않는다. 사람이 bf-execute의 에픽 결과에서 판단한다.

### E4. 완료 — Done 신호

bf-execute에 전달:
- `"done"` + sprint-status.yaml 경로 + review.md 경로
- 종료 (컨텍스트 소멸).

---

## E2E Agent 지침

E2E agent에게 전달할 인라인 지침이다. Agent는 이 지침을 그대로 따른다.

**yq 전제조건 체크** (최초 1회):
```bash
command -v yq >/dev/null 2>&1 || { echo "❌ yq not installed. Install: brew install yq"; exit 1; }
```

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
- **인프라 오류** (regression Story 생성 안함, `"escalation"` + 인프라 오류 사유를 Lead에 보고):
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

- `"done"` + sprint-status.yaml 경로 + review.md 경로 (에픽 실행 완료)
- `"done"` + sprint-status.yaml 경로 + stories/ 경로 (plan 모드 완료)
- 중간 과정은 파일에만 기록, bf-execute 컨텍스트에 남기지 않음
