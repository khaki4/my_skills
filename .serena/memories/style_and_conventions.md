# Style and Conventions

## Skill File Structure
모든 스킬은 `skills/{skill-name}/SKILL.md`에 위치하며 다음 구조를 따른다:
1. YAML frontmatter: `name`, `description` (필수), `disable-model-invocation` (선택)
2. 섹션: Overview, When to Use, Instructions, Output Format
3. `description`은 "Use when..." 으로 시작, 트리거 조건만 기술

## Skill Writing Pattern
- TDD 방식: RED(베이스라인) → GREEN(스킬 작성) → REFACTOR(루프홀 보완)
- 스킬 간 연결은 Lead 계층을 통해 (플랫 체인 아님)

## Model Policy
- `sonnet` = `claude-sonnet-4-6` (최솟값)
- `opus` = `claude-opus-4-6`
- `haiku` 전 스킬에서 사용 금지

## Commit Message Format
- `[{TICKET}] 메시지` (예: `[HACKLE-13554] 로그인 폼 유효성 검사 추가`)

## Sprint Identifier
- Jira 티켓 번호 그대로 사용 (SPRINT-01 같은 순번 대신)

## Document Management
- `docs/` 하위 산출물은 Phase 1-3 동안 git 미관리
- Phase 4 Archive 시점에 비로소 git 관리
- Story agent 커밋은 코드 변경만 포함
