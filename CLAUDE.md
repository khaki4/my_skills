# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains custom Claude Code skills that implement the **BF (Brownfield) Workflow** - a structured development pipeline for brownfield projects that integrates:
- Tech Spec creation and multi-perspective review
- Epic/Story breakdown with difficulty-based execution strategies
- TDD implementation with hierarchical Lead agent orchestration
- Epic-level integration review (OCR + Convention Guard)
- E2E testing and sprint archiving

## Design Principles

1. **Upper agents don't know intermediate details** — Context isolation, pollution prevention
2. **Upper agents maintain project direction** — Coordinate teammates, decide based on project direction
3. **"done" + files on completion** — Context dies, only files persist
4. **Dispute resolution**: Teammates direct talk → Lead mediation → discard (record as unresolved)
5. **Files carry all context** — Modification instructions, review results, stuck reports are all file-based
6. **Single writer per phase** — No concurrent sprint-status.yaml writes; each phase has one writer

## Skill Architecture

All skills are located in `skills/{skill-name}/SKILL.md`. Each skill follows a consistent structure:
- YAML frontmatter with `name` and `description`
- Overview, When to Use, Instructions, and Output Format sections
- Skills chain via Lead hierarchy (not flat chain)

### Lead Skill Hierarchy

The BF workflow uses 4 Lead skills with distinct coordination patterns:

| Lead | Coordination Pattern | Role | Model |
|------|---------------------|------|-------|
| **bf-lead-orchestrate** | Sequence | "done" → branch → trigger next Lead | Always Opus |
| **bf-lead-plan** | Distribute | Epic/Story structure, parallel distribution + collection | Always Opus |
| **bf-lead-implement** | Monitor | Spawn agents + "done"/"stuck" reception + status update | Opus/Sonnet |
| **bf-lead-review** | Discourse | Reviewer debate + Human checkpoint ② + decision delivery | Opus/Sonnet |

Model selection for bf-lead-implement and bf-lead-review: Opus if L/XL stories in epic, Sonnet if S/M only.

### Workflow Sequence

> Full process graph with ASCII diagrams, branching tables, and protocol details: **[BF-WORKFLOW-GRAPH.md](./BF-WORKFLOW-GRAPH.md)**

The BF workflow executes in this sequence:

1. **`/bf-spec`** → Human provides AC + Slack context → Tech Spec creation
2. **`bf-lead-review`** (auto, tech-spec mode) → Agent Teams multi-perspective review
3. **Human approval checkpoint ①** → Approve or request revisions
4. **`/bf-execute`** → Spawns `bf-lead-orchestrate` (context isolation)
5. **`bf-lead-plan`** (auto) → Generate Epic/Story structure with difficulty tags
5a. **Human confirmation** → Review epic/story structure, adjust difficulty tags
6. **Epic Loop** (per epic, sequential):
   - 6a. **`bf-lead-implement`** → TDD implementation with difficulty-based agent strategies
   - 6b. **E2E agent** → Write + execute E2E tests
   - 6c. **`bf-lead-review`** (epic-review mode) → Epic integration review + **Human checkpoint ②**
7. **`/bf-archive-sprint`** → Move docs to archive, update changelog
8. **`/bf-metrics`** → Analyze sprint metrics and suggest optimizations (optional, manual)
9. **`/bf-update-conventions`** → Extract patterns and update convention rules

- **`/bf-resume`** → Resume interrupted workflow (manual, from any point)

### Coordination Patterns

| Pattern | Core Behavior | Used By |
|---------|--------------|---------|
| **Distribute** | Lead splits work → agents execute in parallel → Lead collects | bf-lead-plan, /teams |
| **Monitor** | Lead spawns agents → monitors "done"/"stuck" → updates status | bf-lead-implement, /teams |
| **Discourse** | Independent analysis → cross-verification → consensus/dissent separation | bf-lead-review, /teams |
| **Sequence** | Step-by-step triggering, branching only, no analysis | bf-lead-orchestrate, /teams |

These patterns are also available in the general-purpose `/teams` skill.

## Key Concepts

### Ralph Loop

**Ralph Loop** is an iterative TDD cycle executed by a single story agent without external coordination:
1. Write test based on AC
2. Run test → verify Red (failure)
3. Implement code to pass test
4. Run test → verify Green (success) — **max 5 retries, approach switch on 2 consecutive same errors**
5. Refactor if needed
6. Commit changes

Used for all difficulty levels — the difference is agent composition (S/M: single Sonnet, L: Opus lead + Sonnet implementers, XL: Opus lead + 3+ teammates). Ralph Loop guardrails are defined inline in `skills/bf-lead-implement/SKILL.md`.

### Difficulty-Based Execution Strategies

Each Story is tagged with a difficulty level that determines agent composition:

- **S (Simple)**: Single file, clear AC, no dependencies
  - Execution: Single Sonnet agent
  - Review: Epic-level integration review

- **M (Medium)**: 2-3 files, inter-module connections
  - Execution: Single Sonnet agent
  - Review: Epic-level integration review

- **L (Large)**: Multiple files, significant architectural impact
  - Execution: Sub-Lead (Opus) + Sonnet implementers
  - Review: Epic-level integration review with discourse

- **XL (Complex)**: Cross-layer, security/performance considerations, design decisions
  - Execution: Sub-Lead (Opus) + 3+ teammates
  - Review: Epic-level integration review with extended discourse

**Important**: Story-level review is removed. All review happens at epic level after E2E passes.

### Agent Teams Patterns

- **Main session delegates to orchestrate**: Main context stays minimal (only "done" + file paths)
- **Epic execution is sequential**: Epics run one at a time in dependency order
- **Story execution is parallel**: Stories within an Epic can run concurrently (if no file overlap)
- **Lead never touches code directly**: bf-lead-implement delegates ALL stories to agents
- **sprint-status.yaml single writer**: Only the Lead responsible for current phase writes to it

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
    {EPIC-ID}-review.md
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
```

State field values:
- **status**: `todo` → `in_progress` → `done`
- **tdd**: `pending` → `done`
- **review**: `pending` → `approved`
- **e2e**: `pending` → `passed`

Metric field values (recorded by downstream agents, initialized with defaults):
- **model_used**: `null` → `"sonnet"` | `"opus-lead"` | `"opus-lead+3"` (bf-lead-implement records)
- **ralph_retries**: `0` → Green verification failure retry count (bf-lead-implement records)
- **ralph_approaches**: `0` → Stuck Detection approach switch count (bf-lead-implement records)
- **review_blockers**: `0` → Blocker count (bf-lead-review records)
- **review_recommended**: `0` → Recommended count (bf-lead-review records)
- **failure_tag**: `null` → Failure tag (E2E agent records on regression stories only)
- **is_regression**: `false` → Whether story was auto-generated from E2E failure (E2E agent records)
- **parent_story**: `null` → Source story ID for regression (E2E agent records)
- **ralph_stuck**: `false` → `true` when Ralph Loop limit exceeded (bf-lead-implement records)

### sprint-status.yaml Write Permissions

Each phase is sequential, so no concurrent writes occur. Story agents never touch sprint-status.yaml.

| Agent | Permission | Writes |
|-------|-----------|--------|
| Story agent | None (code + commit only) | — |
| bf-lead-implement | Write | Story status, metrics (retries, approaches, stuck) |
| E2E agent | Write | E2E status, failure tag, regression story addition |
| bf-lead-review | Write | Review status, blocker/recommended counts |
| bf-lead-orchestrate | Phase transition write | Status transitions (in_progress, skipped), difficulty adjustments |

### sprint-status.yaml Update Protocol

When updating sprint-status.yaml, follow this **Read-yq-Verify** protocol:

1. **Read**: Read sprint-status.yaml to confirm current state before modification.
2. **Update**: Use `yq -i` command via Bash tool for programmatic YAML field update.
3. **Verify**: Re-read file after update to confirm changes applied correctly.

**Core rules:**
- **Minimum scope**: Only change fields in your assigned Story/Epic. Never modify other Stories.

**yq prerequisite**: `yq` (Mike Farah's Go-based, v4+) must be installed. Each Lead/E2E agent checks once at startup:
```bash
command -v yq >/dev/null 2>&1 || { echo "❌ yq not installed. Install and retry:"; echo "  macOS: brew install yq"; echo "  Linux: https://github.com/mikefarah/yq#install"; exit 1; }
```

**yq usage examples:**
```bash
# Update story status
yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "done"' docs/sprint-status.yaml

# Update multiple fields at once
yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "done" | .<SPRINT>.<EPIC>.<STORY>.tdd = "done"' docs/sprint-status.yaml

# Add regression story
yq -i '.<SPRINT>.<EPIC>.<NEW-STORY> = {"status":"todo","difficulty":"S","tdd":"pending","review":"pending","model_used":null,"ralph_retries":0,"ralph_approaches":0,"review_blockers":0,"review_recommended":0,"failure_tag":"impl-bug","is_regression":true,"parent_story":"story-1","ralph_stuck":false}' docs/sprint-status.yaml
```

### TDD Cycle Implementation

All stories follow strict TDD:
1. Write unit tests based on AC
2. Verify Red (test fails)
3. Implement code
4. Verify Green (test passes) — max 5 retries with stuck detection
5. Refactor if needed
6. Git commit per story

### Epic Integration Review (OCR + Convention Guard)

Reviews happen at **epic level** (not per-story) after E2E passes:

- **Convention Guard** (mandatory): `docs/conventions.md` compliance check
- **Additional reviewers** (1-2): Architecture, Security, or Performance based on epic scope
- **Discourse pattern**: Independent analysis → cross-verification → consensus/dissent separation
- **Human checkpoint ②**: Human reviews with live reviewer agents, decides on modifications
- Findings categorized as: Blocker, Recommended, Confirmed, Unresolved Disputes

### Dispute Resolution Protocol

Applied across all Lead skills when teammates disagree:
1. **Teammates direct talk**: SendMessage for direct challenge/agree/supplement → consensus reported to Lead
2. **Lead mediation on disagreement**: Lead decides based on project direction (tech-spec, conventions)
3. **Still no consensus → discard (record)**: Include as "unresolved dispute" in final results, stop spending tokens

### Escalation Protocol

"stuck" is a final status, not intermediate. Each layer judges within its scope and passes files upward.

```
Story agent → "stuck" + stuck.md → terminates
bf-lead-implement → continues other stories → reports to orchestrate with sprint-status.yaml + stuck.md
orchestrate → presents to human for decision (AC modification / approach change / skip / redesign)
```

### Metrics and Optimization

`/bf-metrics` analyzes metric data in sprint-status.yaml to **suggest** workflow optimizations (read-only).

- **Suggestions only**: No auto-application of model allocation changes or difficulty re-tagging. Human decides.
- **Execution timing**: After `/bf-archive-sprint`, before `/bf-update-conventions` (optional)
- **Analysis scope**: Current + all archived sprints' completed stories
- **Key analyses**:
  - (difficulty, model_used) pair aggregation (retries, stuck rate, blockers, regression rate)
  - Model allocation optimization suggestions (threshold-based)
  - Difficulty over/under-estimation re-tagging suggestions
  - E2E failure tag pattern analysis
- **Legacy compatibility**: Skips stories from older sprints that lack metric fields

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

- **Human checkpoints are intentional**: The workflow requires human approval at specific points (① Tech Spec review, ② Epic integration review, stuck escalation)
- **Context isolation**: Main session → orchestrate → Leads → agents. Each layer terminates with "done" + files, context dies
- **Files carry all context**: Modification instructions in review.md, stuck reports in stuck.md — no relay through conversation
- **Lead never touches code**: bf-lead-implement delegates ALL difficulty levels to agents
- **Append-only changelog**: CLAUDE.md changelog in target projects is append-only to track sprint history
- **Convention accumulation**: `docs/conventions.md` grows over sprints as patterns are discovered and codified
- **Agent-browser for E2E**: E2E tests use @ref-based element selection from accessibility trees, not CSS selectors
- **Language convention**: This CLAUDE.md uses English for international readability. Individual skill files (`SKILL.md`) use Korean as the primary language, with technical terms in English (e.g., Agent Teams, Ralph Loop, sprint-status.yaml). Mapping: "Human approval checkpoint ①②" (CLAUDE.md) = "사람 개입 ①②" (SKILL.md)
