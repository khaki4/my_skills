# Suggested Commands

## System (Darwin/macOS)
- `git`, `ls`, `cd`, `grep`, `find` — 표준 macOS CLI
- `brew` — 패키지 관리자

## Skill Installation & Testing
```bash
# 스킬 설치 테스트
npx skills add khaki4/my_skills --skill {skill-name} -a claude-code
```

## Git
```bash
git status
git log --oneline -10
git diff
```

## Sprint Status (yq 필요)
```bash
# yq 설치 확인
command -v yq >/dev/null 2>&1 || brew install yq

# sprint-status.yaml 필드 업데이트 예시
yq -i '.<TICKET>.<EPIC>.<STORY>.status = "done"' docs/sprint-status.yaml
```

## Serena
```bash
# 프로젝트 활성화 (MCP 도구로)
# activate_project → check_onboarding_performed
```

## Notes
- 이 저장소는 코드가 아닌 Markdown 스킬 파일 모음이므로 lint/test/build 명령 없음
- 스킬 품질 검증은 실제 프로젝트에서 설치 후 실행하여 확인
