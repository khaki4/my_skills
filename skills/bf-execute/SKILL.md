---
name: bf-execute
description: BF 워크플로우 실행 진입점. bf-lead-orchestrate를 스폰하여 에픽/스토리 생성부터 구현, E2E, 리뷰까지 전체 파이프라인을 실행한다.
---

# BF Execute (Entry Point)

## Overview

BF 워크플로우의 실행 진입점이다. `bf-lead-orchestrate`를 스폰하여 에픽/스토리 생성 → TDD 구현 → E2E 검증 → 통합 리뷰 → 아카이빙까지 전체 파이프라인을 조율한다. 메인 세션은 위임만 하고 컨텍스트를 최소로 유지한다.

## When to Use

- 사용자가 `/bf-execute`를 입력했을 때
- `/bf-spec`으로 Tech Spec이 승인된 후 다음 단계로 진행할 때

## Prerequisites

- 승인된 Tech Spec: `docs/tech-specs/{TICKET}-tech-spec.md`
- Tech Spec 리뷰 통과 (사람 개입 ① 완료)

## Instructions

### 1. 사전 확인

- `docs/tech-specs/{TICKET}-tech-spec.md` 존재 확인
- 사용자에게 Jira 티켓 번호를 확인한다 (미제공 시 요청).

### 2. bf-lead-orchestrate 스폰

- Task tool 사용, `model: opus`
- 전달: tech-spec 경로, conventions.md 경로 (있으면)
- 메인 세션은 `"done"` + sprint-status.yaml 경로만 수신 대기한다.
- 중간 과정은 메인 세션 컨텍스트에 들어오지 않는다 (컨텍스트 격리).

### 3. 완료 수신 후 안내

orchestrate로부터 `"done"` 수신 시:

```
워크플로우가 완료되었습니다.

다음 단계:
1. /bf-archive-sprint — 스프린트 아카이빙
2. /bf-metrics — 메트릭 분석 (선택)
3. /bf-update-conventions — 컨벤션 업데이트
```

## Output Format

- 사용자에게 완료 안내 + 다음 단계 안내
- 모든 산출물은 orchestrate 이하 Lead들이 생성 (메인 세션은 파일을 직접 생성하지 않음)
