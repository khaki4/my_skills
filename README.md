# 전체 설치

npx skills add {your-github-username}/{repo-name}

# 특정 스킬만 설치

npx skills add {your-github-username}/{repo-name} --skill create-tech-spec

# 특정 에이전트 지정 (claude-code, cursor, codex 등)

npx skills add {your-github-username}/{repo-name} -a claude-code

# 글로벌 설치 (모든 프로젝트에서 사용)

npx skills add {your-github-username}/{repo-name} -g

| #   | 스킬                           | 실행 주체                  |
| --- | ------------------------------ | -------------------------- |
| 1   | `/bf-create-tech-spec`         | 사람                       |
| 2   | `/bf-review-tech-spec`         | 자동 (1 이후)              |
| 3   | `/bf-create-epics-and-stories` | 사람 (개입 ① 이후)         |
| 4   | `/bf-create-e2e`               | 자동 (에픽 시작 전)        |
| 5   | `/bf-implement-story`          | 자동 (스토리별)            |
| 6   | `/bf-review-code`              | 자동 (M 이상)              |
| 7   | `/bf-run-e2e`                  | 자동 (스토리 전체 승인 후) |
| 8   | `/bf-archive-sprint`           | 사람                       |
| 9   | `/bf-update-conventions`       | 사람                       |
