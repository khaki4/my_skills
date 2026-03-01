# Project Overview

## Purpose
Claude Code 커스텀 스킬 모음 저장소. **BF (Brownfield) Workflow**를 구현하는 구조화된 개발 파이프라인.
4개 Phase: Spec → Plan → Epic Loop (TDD) → Archive

## Tech Stack
- **언어**: Markdown (스킬 정의 파일)
- **플랫폼**: Claude Code (Anthropic CLI)
- **스킬 설치**: `npx skills add khaki4/my_skills --skill {name} -a claude-code`
- **GitHub**: `khaki4/my_skills`

## Repository Structure
```
CLAUDE.md                    # 프로젝트 지침 (핵심)
BF-WORKFLOW-GRAPH.md         # Phase 상세 및 Sub-Phase 계약
REFERENCES.md                # 참고 기술 및 방법론
README.md                    # 공개 문서
skills/                      # 스킬 디렉토리 (각 스킬은 SKILL.md 포함)
  bf-spec/                   # Phase 1: AC → Tech Spec
  bf-execute/                # 사람 경계 (워크플로우 실행)
  bf-lead-orchestrate/       # Sequence 패턴 조율
  bf-lead-plan/              # Distribute 패턴 (Epic/Story 분해)
  bf-lead-implement/         # Monitor 패턴 (TDD 구현)
  bf-lead-review/            # Discourse 패턴 (통합 리뷰)
  bf-resume/                 # 중단 복구
  bf-archive-sprint/         # Phase 4: 아카이빙
  bf-metrics/                # 메트릭 분석
  bf-update-conventions/     # 컨벤션 규칙 업데이트
  teams/                     # 범용 팀 조율 스킬
  d/                         # 영한 개발자 사전
docs/                        # 산출물 (아카이브 전 git 미관리)
.claude/                     # Claude Code 설정
```

## Key Concepts
- **Ralph Loop**: TDD 사이클 (Red → Green → Refactor), 최대 5회 재시도
- **Lead 계층**: orchestrate → plan/implement/review (각각 다른 조율 패턴)
- **컨텍스트 격리**: 상위 에이전트는 중간 과정을 모르고, "done" + 파일만 전달
- **파일 기반 소통**: 모든 전달은 파일 (modification.md, review.md, stuck.md)
- **사람 판단 3회**: Spec 초안 검토, Spec 최종 승인, Epic 결과 확인
