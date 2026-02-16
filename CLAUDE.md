# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains custom Claude Code skills that implement the **BF (Brownfield) Workflow** - a structured development pipeline for brownfield projects that integrates:
- Tech Spec creation and multi-perspective review
- Epic/Story breakdown with difficulty-based execution strategies
- TDD implementation with Agent Teams orchestration
- Open Code Review (OCR) and Convention Guard
- E2E testing and sprint archiving

## Skill Architecture

All skills are located in `skills/{skill-name}/SKILL.md`. Each skill follows a consistent structure:
- YAML frontmatter with `name` and `description`
- Overview, When to Use, Instructions, and Output Format sections
- Skills automatically chain together where appropriate

### Workflow Sequence

The BF workflow executes in this sequence:

1. **`/bf-create-tech-spec`** → Human provides AC + Slack context
2. **`/bf-review-tech-spec`** → Agent Teams multi-perspective review (auto)
3. **Human approval checkpoint ①** → Approve or request revisions
4. **`/bf-create-epics-and-stories`** → Generate Epic/Story structure with difficulty tags
4a. **Human confirmation** → Review epic/story structure, adjust difficulty tags if needed
5. **`/bf-create-e2e`** → Write E2E tests before implementation (auto, per epic)
6. **`/bf-implement-story`** → TDD implementation with difficulty-based strategies (auto, per story)
7. **`/bf-review-code`** → OCR + Convention Guard review (auto, for M+ difficulty)
8. **Human approval checkpoint ②** → Approve or request revisions per story
9. **`/bf-run-e2e`** → Execute E2E tests after all stories approved (auto)
10. **`/bf-archive-sprint`** → Move docs to archive, update changelog
11. **`/bf-metrics`** → Analyze sprint metrics and suggest optimizations (optional, manual)
12. **`/bf-update-conventions`** → Extract patterns and update convention rules

- **`/bf-resume`** → 중단된 워크플로우 복구 (어느 시점에서든 수동 실행 가능)

## Key Concepts

### Ralph Loop

**Ralph Loop** is an iterative TDD cycle executed by a single agent without external coordination:
1. Write test based on AC
2. Run test → verify Red (failure)
3. Implement code to pass test
4. Run test → verify Green (success) — **최대 5회 재시도, 동일 에러 2연속 시 접근 전환**
5. Refactor if needed
6. Commit changes

Used for S and M difficulty stories where a single agent (typically Sonnet) can complete the work independently without requiring Agent Teams coordination. 무한 루프 방지를 위한 가드레일은 `skills/bf-implement-story/SKILL.md`의 "Ralph Loop 가드레일" 섹션 참조.

### Difficulty-Based Execution Strategies

Each Story is tagged with a difficulty level that determines execution approach:

- **S (Simple)**: Single file, clear AC, no dependencies
  - Execution: ralph-loop with Sonnet only
  - Review: None (direct human approval)

- **M (Medium)**: 2-3 files, inter-module connections
  - Execution: ralph-loop with Sonnet
  - Review: OCR with Opus

- **L (Large)**: Multiple files, significant architectural impact
  - Execution: Agent Teams (Opus Lead + Sonnet implementer)
  - Review: Agent Teams discourse (multi-reviewer debate)

- **XL (Complex)**: Cross-layer, security/performance considerations, design decisions
  - Execution: Agent Teams with 3+ teammates
  - Review: Agent Teams discourse with extended debate

### Agent Teams Patterns

- **Main session delegates to Lead**: Main context stays clean, Lead coordinates teammates
- **Epic execution is sequential**: Epics run one at a time in dependency order
- **Story execution is parallel**: Stories within an Epic can run concurrently
- **Model selection by complexity**: Simple→Sonnet, Complex→Opus

### File Structure Conventions

When using these skills, the following directory structure will be created in the target project:

```
docs/
  tech-specs/
    {TICKET}-tech-spec.md
  stories/
    {TICKET}-story-{N}.md
  reviews/
    {TICKET}-tech-spec-review.md
    {STORY-ID}-review.md
  sprint-status.yaml
  conventions.md          # Convention Guard rules
  archive/
    {SPRINT-XX}/
      tech-specs/
      stories/
      reviews/
      sprint-status.yaml

tests/
  e2e/
    {epic-name}/
      {scenario-name}.sh
```

### sprint-status.yaml Structure

Tracks all Epic/Story progress:

```yaml
SPRINT-XX:
  epic-1:
    story-1:
      status: todo
      difficulty: S
      tdd: pending
      review: pending
      model_used: null
      ralph_retries: 0
      ralph_approaches: 0
      review_blockers: 0
      review_recommended: 0
      failure_tag: null
      is_regression: false
      parent_story: null
      ralph_stuck: false
    story-2:
      status: todo
      difficulty: M
      tdd: pending
      review: pending
      model_used: null
      ralph_retries: 0
      ralph_approaches: 0
      review_blockers: 0
      review_recommended: 0
      failure_tag: null
      is_regression: false
      parent_story: null
      ralph_stuck: false
    e2e: pending
  epic-2:
    story-3:
      status: todo
      difficulty: L
      tdd: pending
      review: pending
      model_used: null
      ralph_retries: 0
      ralph_approaches: 0
      review_blockers: 0
      review_recommended: 0
      failure_tag: null
      is_regression: false
      parent_story: null
      ralph_stuck: false
    e2e: pending
```

State field values:
- **status**: `todo` → `in_progress` → `done`
- **tdd**: `pending` → `done`
- **review**: `pending` → `approved` (S difficulty skips to `approved` directly)
- **e2e**: `pending` → `written` → `passed`

Metric field values (recorded by downstream skills, initialized with defaults):
- **model_used**: `null` → `"sonnet"` | `"opus-lead"` | `"opus-lead+3"` (bf-implement-story가 기록)
- **ralph_retries**: `0` → Green 검증 실패 재시도 횟수 (bf-implement-story가 기록)
- **ralph_approaches**: `0` → Stuck Detection 접근 전환 횟수 (bf-implement-story가 기록)
- **review_blockers**: `0` → 🔴 Blocker 건수 (bf-review-code가 기록)
- **review_recommended**: `0` → 🟡 Recommended 건수 (bf-review-code가 기록)
- **failure_tag**: `null` → 실패 태그 (bf-run-e2e가 regression Story에만 기록)
- **is_regression**: `false` → E2E 실패로 자동 생성된 Story 여부 (bf-run-e2e가 기록)
- **parent_story**: `null` → regression일 때 원인 Story ID (bf-run-e2e가 기록)
- **ralph_stuck**: `false` → Ralph Loop 한도 초과 시 `true` (bf-implement-story가 기록)

### sprint-status.yaml 업데이트 프로토콜

sprint-status.yaml은 여러 스킬(bf-implement-story, bf-create-e2e, bf-review-code, bf-run-e2e)이 병렬로 수정할 수 있다. 다음 **Read-Merge-Write with Retry** 프로토콜을 반드시 따른다:

1. **Read**: 수정 직전에 sprint-status.yaml을 읽는다 (캐시된 내용 사용 금지).
2. **Merge**: 자신이 변경하려는 Story/에픽 블록**만** 수정하고, 나머지 블록은 읽은 그대로 보존한다.
3. **Write**: Edit 도구로 해당 Story 블록의 `old_string` → `new_string` 치환을 수행한다.
4. **Verify**: Write 직후 파일을 다시 읽어서 자신의 변경이 정상 반영되었는지 확인한다.
5. **Retry**: 검증 실패 시 (다른 Agent가 동시에 덮어쓴 경우) step 1부터 최대 **3회** 재시도한다. 3회 초과 시 사용자에게 알린다.

**핵심 원칙:**
- **전체 파일 재생성 금지**: Write 도구로 전체 파일을 덮어쓰지 않는다. 반드시 Edit 도구로 해당 블록만 치환한다.
- **최소 범위 수정**: 자신이 담당하는 Story의 필드만 변경한다. 다른 Story 필드를 절대 수정하지 않는다.
- **Read-Write 간격 최소화**: Read와 Write 사이에 불필요한 작업을 끼우지 않는다.

> 이 프로토콜은 진정한 Optimistic Concurrency (version-based)가 아닌 **Best-Effort Merge** 패턴이다. Claude Code의 Edit 도구가 text-based 치환이므로 동시 쓰기 시 이론적 데이터 손실 가능성이 있으나, 블록 단위 치환 + 검증 + 재시도로 실질적 위험을 최소화한다.

### TDD Cycle Implementation

All stories follow strict TDD:
1. Write unit tests based on AC
2. Verify Red (test fails)
3. Implement code
4. Verify Green (test passes)
5. Refactor if needed
6. Git commit per story

### OCR (Open Code Review) and Convention Guard

- **OCR**: Multi-perspective code review by Agent Teams
- **Convention Guard**: Automated enforcement of `docs/conventions.md` rules
- Reviews check: convention compliance, duplication, test coverage, security, code quality
- Findings categorized as: 🔴 Blocker, 🟡 Recommended, 🟢 Confirmed

### Metrics and Optimization

`/bf-metrics`는 sprint-status.yaml에 기록된 메트릭 데이터를 분석하여 워크플로우 최적화를 **제안**하는 read-only 스킬이다.

- **제안만 제공**: 모델 배당 변경이나 난이도 재태깅을 자동 적용하지 않음. 사람이 판단.
- **실행 시점**: `/bf-archive-sprint` 후, `/bf-update-conventions` 전에 선택적으로 실행
- **분석 범위**: 현재 + 아카이브된 모든 스프린트의 완료된 Story
- **주요 분석**:
  - (difficulty, model_used) 페어별 집계 (retries, stuck rate, blockers, regression rate)
  - 모델 배당 최적화 제안 (임계값 기반)
  - 난이도 과소/과대평가 재태깅 제안
  - E2E 실패 태그 패턴 분석
- **레거시 호환**: 메트릭 필드 없는 이전 스프린트 Story는 건너뜀

## Adding New Skills

When creating new skills in this repository:

1. Create directory: `skills/{skill-name}/`
2. Add `SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: skill-name
   description: Brief description for skill discovery
   ---
   ```
3. Include these sections: Overview, When to Use, Instructions, Output Format
4. Update the workflow sequence in this CLAUDE.md if the skill integrates into the pipeline
5. Test the skill with `npx skills add {username}/my-skills --skill {skill-name} -a claude-code`

## Important Notes

- **Human checkpoints are intentional**: The workflow requires human approval at specific points to validate direction before proceeding
- **Context preservation**: Main session delegates to Agent Teams to keep context clean for long workflows
- **Append-only changelog**: CLAUDE.md changelog in target projects is append-only to track sprint history
- **Convention accumulation**: `docs/conventions.md` grows over sprints as patterns are discovered and codified
- **Agent-browser for E2E**: E2E tests use @ref-based element selection from accessibility trees, not CSS selectors
