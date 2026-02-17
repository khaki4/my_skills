---
name: bf-spec
description: AC와 Slack 논의를 입력받아 Tech Spec을 작성하고, 자동으로 bf-lead-review를 통해 다관점 리뷰를 수행한다. BF 워크플로우 진입점.
---

# BF Spec (Entry Point)

## Overview

BF 워크플로우의 진입점이다. 기획자가 제공한 AC와 Slack 논의를 기반으로 Tech Spec을 작성하고, 자동으로 `bf-lead-review`를 스폰하여 다관점 리뷰를 수행한다. 리뷰 결과를 사람에게 제시하여 승인 여부를 판단하게 한다 (사람 개입 ①).

## When to Use

- 사용자가 `/bf-spec`을 입력했을 때
- 새로운 기능 개발 또는 변경 요청이 있을 때

## Prerequisites

- 워크플로우 진입점이므로 별도 전제조건 없음
- 사용자가 기획자 AC 또는 변경 요청 내용을 준비한 상태여야 함

## Instructions

### 1. 입력 수집

사용자에게 다음 입력을 요청한다:
- 기획자 AC 문서 또는 내용
- Slack 논의 요약 (있으면)
- 관련 Jira 티켓 번호

### 2. 코드베이스 분석

기존 코드베이스를 분석한다:
- 변경 대상 모듈/파일 식별
- 기존 아키텍처 패턴 확인
- 의존성 그래프 파악

### 3. Tech Spec 작성

Tech Spec 문서를 작성한다:
- 변경 목적 및 배경
- 현재 상태 (As-Is)
- 목표 상태 (To-Be)
- 변경 영향 범위
- AC (기획자 AC 그대로 포함)
- 기술 제약사항 및 리스크
- 테스트 전략 개요

### 4. 저장 및 커밋

- `docs/tech-specs/{TICKET}-tech-spec.md`에 저장한다.
- `docs/tech-specs/` 디렉토리가 없으면 생성한다.
- git commit: `docs({TICKET}): create tech spec`

### 5. bf-lead-review 자동 스폰 (Tech-Spec 모드)

- `bf-lead-review`를 스폰한다:
  - Task tool 사용, `model: opus`
  - 파라미터: `mode: "tech-spec"`, tech-spec 경로 전달
- 메인 세션은 `"done"` + review.md 경로만 수신한다 (컨텍스트 격리).

### 6. 리뷰 결과 제시 및 사람 개입 ①

- review.md를 읽어서 사람에게 제시한다.
- 사람의 결정:
  - **승인** → "`Tech Spec이 승인되었습니다. /bf-execute로 구현을 시작하세요.`" 안내
  - **수정 요청** → Tech Spec 수정 후 5단계(bf-lead-review 스폰)를 재실행

## Output Format

- `docs/tech-specs/{TICKET}-tech-spec.md` — Tech Spec 문서
- `docs/reviews/{TICKET}-tech-spec-review.md` — 리뷰 결과 (bf-lead-review가 생성)

마크다운 형식. 섹션: Background, As-Is, To-Be, Impact Analysis, Acceptance Criteria, Technical Constraints, Test Strategy.
