---
name: d
description: Use when the user types /d followed by an English word or phrase to get a Korean translation with developer context
---

# English-Korean Developer Dictionary

## Overview

Translate English words/phrases to Korean with optional developer context. Uses haiku subagent for minimal cost.

## Instructions

1. Extract the word or phrase from user input (the args after `/d`)
2. Spawn a single Agent with `model: "haiku"` and `subagent_type: "general-purpose"`
3. Pass the prompt below to the agent, replacing `{INPUT}` with the user's word/phrase
4. Output the agent's response directly to the user. Do NOT add commentary before or after.

## Agent Prompt

```
Translate the English word/phrase "{INPUT}" to Korean. Follow this EXACT format with no deviations:

**{INPUT}** /{pronunciation}/
{Korean meanings, comma-separated, one line}

💻 개발 맥락:
- "{INPUT} + common collocation" → Korean explanation
- "another collocation" → Korean explanation

RULES:
- Line 1: Bold word + IPA pronunciation in slashes
- Line 2: Korean core meanings only. One line, comma-separated. No English.
- Line 3+: Dev context section. 2-3 collocations used in code, commits, or docs with Korean explanations.
- ONLY include dev context if the word is a TECHNICAL TERM used in programming, APIs, architecture, or CS theory (e.g., "idempotent", "resilient", "deprecated"). For everyday English words, idioms, or phrases that are merely USED BY developers in conversation (e.g., "break the ice", "on the same page"), OMIT the dev context section entirely. Just output lines 1-2.
- Do NOT add headings, bullet explanations, examples sentences, or any extra content.
- Total output must be under 6 lines.
- Respond in Korean except for the English word itself and collocations.
```

## Output Format

For dev-relevant words:
```
**resilient** /rɪˈzɪliənt/
회복력 있는, 탄력 있는

💻 개발 맥락:
- "resilient system" → 장애 복구가 빠른 시스템
- "resilient to failures" → 실패에 강한
```

For general words with no dev relevance:
```
**break the ice** /breɪk ðə aɪs/
어색한 분위기를 깨다, 서먹함을 없애다
```
