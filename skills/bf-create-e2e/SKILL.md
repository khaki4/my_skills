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

- Story 파일들 존재: `docs/stories/{TICKET}-story-*.md`
- sprint-status.yaml 존재 및 해당 에픽 정의됨
- `tests/e2e/` 디렉토리 존재 (없으면 생성)

## Instructions

1. 메인 세션에서 직접 수행한다 (Sonnet이면 충분. AC 읽고 shell script 생성하는 반복 성격 작업).

2. 대상 에픽의 모든 Story와 AC를 읽는다.

3. E2E 테스트 시나리오를 도출한다:
   - 각 AC를 E2E 관점에서 시나리오화
   - Happy path + 주요 실패 경로 포함
   - 사용자 플로우 기준으로 시나리오 순서 결정

4. agent-browser CLI 기반 테스트 스크립트를 작성한다:
   - `snapshot` → `@ref` 또는 `find` semantic locator로 요소 선택
   - CSS 셀렉터 사용 금지 (DOM 변경에 취약)
   - 각 시나리오별 독립 실행 가능한 구조

5. 테스트 스크립트 파일을 저장한다:
   - `tests/e2e/{epic-name}/` 디렉토리에 저장
   - 파일명: `{scenario-name}.sh` (shell 스크립트)
   - agent-browser는 CLI 도구이므로 shell script로 순차 호출한다 (`.test.ts`가 아님)
   - 테스트 러너(Vitest/Jest) 없이 `zsh`로 직접 실행

6. sprint-status.yaml의 해당 에픽 e2e 상태를 `written`으로 업데이트한다.
   - 업데이트 직전에 파일을 다시 읽어서 최신 상태 확인 (동시성 주의)
   - read → modify → write 간격을 최소화한다

7. 자동으로 해당 에픽의 Story 구현을 시작한다:
   - sprint-status.yaml에서 현재 에픽의 모든 Story 목록을 읽는다
   - 각 Story에 대해 `/bf-implement-story {story-id}`를 병렬로 트리거한다
   - Story 간 의존성이 있으면 순차 실행한다

## Output Format

- `tests/e2e/{epic-name}/*.sh` — E2E 테스트 스크립트 (agent-browser CLI)
- sprint-status.yaml 업데이트

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
