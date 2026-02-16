---
name: bf-create-e2e
description: 에픽 단위로 E2E 테스트 케이스를 선행 작성한다. agent-browser 기반으로 접근성 트리의 @ref를 활용한 테스트 코드를 생성한다.
---

# Create E2E

## Overview

에픽의 Story 구현에 앞서, 해당 에픽의 E2E 테스트 케이스를 선행 작성한다. agent-browser를 사용하여 실제 브라우저 환경에서의 통합 검증을 준비한다.

## When to Use

- 에픽 구현 시작 전 자동 실행
- 사용자가 `/bf-create-e2e`를 직접 입력했을 때

## Prerequisites

- Story 파일들 존재: `docs/stories/{TICKET}-story-*.md` — 미존재 시 실행 중단 및 `/bf-create-epics-and-stories` 선행 실행 안내
- sprint-status.yaml 존재 및 해당 에픽 정의됨 — 미존재 시 실행 중단 및 안내
- `tests/e2e/` 디렉토리 존재 (없으면 생성)

## Instructions

1. 메인 세션에서 직접 수행한다 (단순 반복 작업이므로 Agent Teams 위임 없이 직접 처리. 컨텍스트 오버헤드가 E2E 작성보다 큼).

2. **프로젝트 E2E 타입을 판별한다:**
   프로젝트 루트를 분석하여 아래 기준으로 타입을 결정한다:
   - **브라우저 UI 프로젝트** — 다음 중 하나 이상 해당: `package.json`에 React/Vue/Angular/Svelte/Next.js dependency 존재, `public/index.html` 또는 `src/App.*` 파일 존재, `next.config.*`/`vite.config.*`/`angular.json` 존재
     → agent-browser CLI 기반 E2E 작성 (step 5a)
   - **API-only 프로젝트** — 프론트엔드 프레임워크 없이 Express/Fastify/NestJS/Django/Flask 등 백엔드 프레임워크만 존재, 또는 REST/GraphQL 엔드포인트만 제공
     → curl/httpie 기반 API E2E 작성 (step 5b)
   - **CLI 도구 프로젝트** — `package.json`의 `bin` 필드 존재, 또는 실행 가능한 CLI 엔트리포인트가 있는 프로젝트
     → shell script 기반 CLI E2E 작성 (step 5c)
   - **E2E 불가 프로젝트** — 라이브러리, 유틸리티 패키지, 순수 SDK 등 사용자 인터페이스 없음
     → E2E를 skip하고 sprint-status.yaml의 e2e를 바로 `passed`로 설정. 사용자에게 skip 사유를 안내한다.

3. 대상 에픽의 모든 Story와 AC를 읽는다.
   - **Story 0개 에픽인 경우**: sprint-status.yaml에 해당 에픽의 Story가 없으면 E2E 작성을 skip한다. e2e를 바로 `passed`로 설정하고 다음 에픽으로 진행한다.

4. E2E 테스트 시나리오를 도출한다:
   - 각 AC를 E2E 관점에서 시나리오화
   - Happy path + 주요 실패 경로 포함
   - 사용자 플로우 기준으로 시나리오 순서 결정

5. 프로젝트 E2E 타입에 따라 테스트 스크립트를 작성한다:

   **a) 브라우저 UI 프로젝트 — agent-browser CLI 기반:**
   - `snapshot` → `@ref` 또는 `find` semantic locator로 요소 선택
   - CSS 셀렉터 사용 금지 (DOM 변경에 취약)
   - 각 시나리오별 독립 실행 가능한 구조

   **b) API-only 프로젝트 — curl/httpie 기반:**
   - API 엔드포인트를 curl 또는 httpie로 호출
   - 응답 status code, JSON body를 jq로 검증
   - 인증이 필요하면 토큰 발급 → 헤더 설정 포함
   - 예: `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/users | grep 200`

   **c) CLI 도구 프로젝트 — shell command 기반:**
   - CLI 명령어를 실행하고 exit code, stdout, stderr를 검증
   - 예: `./cli --version | grep "1.0.0"`, `./cli process input.txt && diff output.txt expected.txt`

6. 테스트 스크립트 파일을 저장한다:
   - `tests/e2e/{epic-name}/` 디렉토리에 저장
   - 파일명: `{scenario-name}.sh` (shell 스크립트)
   - 모든 E2E 타입(agent-browser, curl, CLI)은 shell script로 통일하여 `zsh`로 직접 실행

7. sprint-status.yaml의 해당 에픽 e2e 상태를 `written`으로 업데이트한다 — **CLAUDE.md의 "sprint-status.yaml 업데이트 프로토콜"을 따른다**.

8. git commit을 수행한다:
   - 메시지: `test({epic-name}): create e2e test scripts`
   - E2E 스크립트 파일 + sprint-status.yaml 변경을 포함한다
   - 세션 중단 시 작업 유실 방지를 위해 반드시 커밋한다

9. 자동으로 해당 에픽의 Story 구현을 시작한다:
   - sprint-status.yaml에서 현재 에픽의 모든 Story 목록을 읽는다
   - **파일 겹침 검사**: 각 Story의 Technical Notes(변경 대상 파일/모듈)를 확인하여 Story 간 수정 파일이 겹치는지 검사한다
     - 겹치는 파일이 없는 Story들: 병렬로 `/bf-implement-story {story-id}`를 트리거한다
     - 겹치는 파일이 있는 Story들: 순차적으로 실행한다 (선행 Story 완료 후 후행 Story 시작)
   - Story 간 의존성(Dependencies 섹션)이 있으면 순차 실행한다

## Output Format

- `tests/e2e/{epic-name}/*.sh` — E2E 테스트 스크립트 (agent-browser CLI)
- sprint-status.yaml 업데이트
- git commit

### E2E 테스트 스크립트 예제 (agent-browser CLI)

```zsh
#!/bin/zsh
# tests/e2e/user-authentication/login-flow.sh
# Test: User can log in with valid credentials

set -e  # 실패 시 중단

# 세션 격리 (병렬 실행 대비)
SESSION="e2e-login-$(date +%s)"

# 1. 로그인 페이지 이동
agent-browser --session $SESSION open http://localhost:3000/login

# 2. 접근성 트리 확인 (디버깅용)
agent-browser --session $SESSION snapshot

# 3. semantic locator로 요소 찾아 입력
agent-browser --session $SESSION find label "Email" fill "user@example.com"
agent-browser --session $SESSION find label "Password" fill "password123"

# 4. 로그인 버튼 클릭
agent-browser --session $SESSION find role button --name "Log In" click

# 5. 대시보드 로딩 대기
agent-browser --session $SESSION wait --text "Dashboard"

# 6. 결과 검증
agent-browser --session $SESSION find role heading --name "Dashboard" is visible
agent-browser --session $SESSION find text "Welcome, user@example.com" is visible

# 7. 증거 수집
agent-browser --session $SESSION screenshot tests/e2e/user-authentication/login-success.png

echo "✅ PASS: login-flow"
```

```zsh
#!/bin/zsh
# tests/e2e/user-authentication/login-error.sh
# Test: User sees error with invalid credentials

set -e

SESSION="e2e-login-error-$(date +%s)"

agent-browser --session $SESSION open http://localhost:3000/login
agent-browser --session $SESSION find label "Email" fill "wrong@example.com"
agent-browser --session $SESSION find label "Password" fill "wrongpass"
agent-browser --session $SESSION find role button --name "Log In" click

# 에러 메시지 확인
agent-browser --session $SESSION wait --text "Invalid credentials"
agent-browser --session $SESSION find role alert is visible

echo "✅ PASS: login-error"
```

### agent-browser 요소 선택 원칙

**✅ 사용해야 하는 방법:**
- **Semantic locator**: `find role button --name "Submit"`, `find label "Email"`, `find text "Sign In"`, `find placeholder "Username"`
- **@ref (snapshot 기반)**: `snapshot`으로 접근성 트리 조회 후 `@e2`, `@e5` 등으로 직접 선택
- **wait 전략**: `wait --text "Welcome"`, `wait --url "**/dashboard"`, `wait --load networkidle`

**❌ 사용 금지:**
- CSS 셀렉터: `#email-input`, `.login-form`, `input[name='email']`
- XPath
- DOM 구현 세부사항에 의존하는 선택자
