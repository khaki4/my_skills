---
name: bf-create-tech-spec
description: 기획자 AC와 Slack 논의 내용을 입력받아 브라운필드 프로젝트의 Tech Spec 문서를 생성한다. 기존 코드베이스를 분석하고 변경 영향도, AC, 기술 제약사항을 포함한 명세를 작성할 때 사용한다.
---

# Create Tech Spec

## Overview

기획자가 제공한 AC(Acceptance Criteria)와 Slack 논의 내용을 기반으로, 현재 코드베이스를 분석하여 Tech Spec 문서를 생성한다.

## When to Use

- 사용자가 `/create-tech-spec`을 입력했을 때
- 새로운 기능 개발 또는 변경 요청이 있을 때
- 기획자의 AC가 준비된 상태에서 기술 명세가 필요할 때

## Instructions

1. 사용자에게 다음 입력을 요청한다:
   - 기획자 AC 문서 또는 내용
   - Slack 논의 요약 (있으면)
   - 관련 Jira 티켓 번호

2. 기존 코드베이스를 분석한다:
   - 변경 대상 모듈/파일 식별
   - 기존 아키텍처 패턴 확인
   - 의존성 그래프 파악

3. Tech Spec 문서를 작성한다:
   - 변경 목적 및 배경
   - 현재 상태 (As-Is)
   - 목표 상태 (To-Be)
   - 변경 영향 범위
   - AC (기획자 AC 그대로 포함)
   - 기술 제약사항 및 리스크
   - 테스트 전략 개요

4. 문서를 `docs/tech-specs/{TICKET-NUMBER}-tech-spec.md`에 저장한다.

5. 완료 후 자동으로 `/review-tech-spec`을 실행한다.

## Output Format

docs/tech-specs/{TICKET-NUMBER}-tech-spec.md

마크다운 형식. 섹션: Background, As-Is, To-Be, Impact Analysis, Acceptance Criteria, Technical Constraints, Test Strategy.
