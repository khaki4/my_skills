---
name: bf-update-conventions
description: 스프린트 중 코드 리뷰에서 발견된 반복 패턴을 conventions.md와 OCR 룰에 반영한다. 자기 강화 루프의 축적 단계로, 다음 스프린트의 리뷰 품질을 높인다.
---

# Update Conventions

## Overview

스프린트 완료 후, 코드 리뷰에서 발견된 패턴과 교훈을 docs/conventions.md와 OCR 룰에 축적한다.

## When to Use

- 사용자가 `/bf-update-conventions`를 입력했을 때
- `/bf-archive-sprint` 완료 후

## Prerequisites

- 아카이브된 스프린트 존재: `docs/archive/{SPRINT-XX}/`
- 아카이브 내 stories, tech-specs 디렉토리 존재
- docs/conventions.md (없으면 신규 생성)

## Instructions

1. 아카이브된 스프린트의 리뷰 이력을 분석한다:
   - 반복적으로 지적된 패턴 추출
   - 블로커로 분류된 이슈 유형 정리
   - Convention Guard가 놓친 패턴 식별

2. 사용자에게 발견된 패턴을 제시한다:
   - 각 패턴의 발생 빈도
   - 대표 사례
   - 제안하는 룰 내용

3. 사용자 승인 후 다음을 업데이트한다:
   - **docs/conventions.md**: 새 컨벤션 룰 추가
   - **OCR 룰**: Convention Guard 리뷰어에 새 체크 항목 추가
   - **CLAUDE.md**: 필요 시 고정 영역에 핵심 규칙 반영

4. git commit을 수행한다:
   - 메시지: `docs: update conventions from sprint {SPRINT-XX}`

## Output Format

- docs/conventions.md 업데이트
- OCR 룰 업데이트
- git commit
