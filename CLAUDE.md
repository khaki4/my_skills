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
5. **`/bf-create-e2e`** → Write E2E tests before implementation (auto, per epic)
6. **`/bf-implement-story`** → TDD implementation with difficulty-based strategies (auto, per story)
7. **`/bf-review-code`** → OCR + Convention Guard review (auto, for M+ difficulty)
8. **Human approval checkpoint ②** → Approve or request revisions per story
9. **`/bf-run-e2e`** → Execute E2E tests after all stories approved (auto)
10. **`/bf-archive-sprint`** → Move docs to archive, update changelog
11. **`/bf-update-conventions`** → Extract patterns and update convention rules

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
    {TICKET-NUMBER}-tech-spec.md
  stories/
    {TICKET}-story-{N}.md
  sprint-status.yaml
  conventions.md          # Convention Guard rules
  archive/
    {SPRINT-XX}/
      tech-specs/
      stories/
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
    story-1: { status: todo, difficulty: S, tdd: pending, review: pending }
    story-2: { status: todo, difficulty: M, tdd: pending, review: pending }
    e2e: pending
  epic-2:
    story-3: { status: todo, difficulty: L, tdd: pending, review: pending }
    e2e: pending
```

State field values:
- **status**: `todo` → `in_progress` → `done`
- **tdd**: `pending` → `done`
- **review**: `pending` → `approved` (S difficulty skips to `approved` directly)
- **e2e**: `pending` → `written` → `passed`

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
