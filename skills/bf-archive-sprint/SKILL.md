---
name: archive-sprint
description: 스프린트 내 모든 에픽이 완료된 후 스프린트 문서를 아카이빙한다. 문서 이동, changelog 업데이트, git commit을 자동으로 수행한다.
---

# Archive Sprint

## Overview

스프린트가 완료되면 관련 문서를 아카이빙하고, changelog를 업데이트하고, git commit을 수행한다.

## When to Use

- 사용자가 `/archive-sprint`를 입력했을 때
- 스프린트 내 모든 에픽의 E2E가 통과한 후

## Instructions

1. sprint-status.yaml을 확인한다:
   - 모든 에픽의 e2e가 `passed`인지 검증
   - 미완료 에픽이 있으면 실행 중단 및 안내

2. 문서를 아카이브 디렉토리로 이동한다:
   - `docs/stories/` → `docs/archive/{SPRINT-XX}/stories/`
   - `docs/tech-specs/` → `docs/archive/{SPRINT-XX}/tech-specs/`
   - `docs/sprint-status.yaml` → `docs/archive/{SPRINT-XX}/sprint-status.yaml`

3. CLAUDE.md의 changelog에 append-only로 기록한다:
   - 스프린트 번호
   - 완료된 에픽/스토리 요약
   - 아카이브 경로 링크

4. git commit을 수행한다:
   - 메시지: `chore: archive sprint {SPRINT-XX}`
   - 아카이브된 파일 + changelog 변경 포함

5. 완료 후 `/update-conventions` 실행을 안내한다.

## Output Format

- `docs/archive/{SPRINT-XX}/` — 아카이브 디렉토리
- CLAUDE.md changelog 업데이트
- git commit
