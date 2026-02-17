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
- `docs/archive/{SPRINT-XX}/reviews/` 디렉토리 존재 (리뷰 결과 파일) — 미존재 시 git log에서 리뷰 관련 커밋 히스토리를 대안으로 분석
- docs/conventions.md (없으면 신규 생성)

## Instructions

1. 아카이브된 스프린트의 리뷰 이력을 분석한다:
   - **1차 소스**: `docs/archive/{SPRINT-XX}/reviews/*.md` 파일들을 읽는다
   - **2차 소스** (리뷰 파일 미존재 시): `git log`에서 `fix:` 커밋 메시지와 변경 패턴을 분석하여 반복 지적 패턴을 추론한다
   - 반복적으로 지적된 패턴 추출
   - 블로커로 분류된 이슈 유형 정리
   - Convention Guard가 놓친 패턴 식별

2. 사용자에게 발견된 패턴을 제시한다:
   - 각 패턴의 발생 빈도
   - 대표 사례
   - 제안하는 룰 내용

3. 사용자 승인 후 다음을 업데이트한다:
   - **docs/conventions.md**: 새 컨벤션 룰 추가. 이 파일이 Convention Guard(OCR 리뷰)의 단일 규칙 소스이므로, 새 체크 항목도 이 파일에 추가한다. **기존 룰은 삭제하지 않는다** (append-only). 기존 룰 보완·구체화만 허용한다.
   - **CLAUDE.md**: 필요 시 고정 영역에 핵심 규칙 반영

4. git commit을 수행한다:
   - 메시지: `docs({SPRINT-XX}): update conventions`

## Output Format

- docs/conventions.md 업데이트 (Convention Guard 규칙 포함)
- CLAUDE.md 업데이트 (필요 시)
- git commit
