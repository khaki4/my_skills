# BF (Brownfield) Workflow

브라운필드 프로젝트를 위한 구조화된 개발 파이프라인. Claude Code 커스텀 스킬로 구현.

## Phase 구조

각 Phase는 독립적인 입력-출력 계약을 가지며, 내부 구현을 다른 Phase에 영향 없이 변경할 수 있다.

```
AC 문서 → [Phase 1. Spec] → [Phase 2. Plan] → [Phase 3. Epic Loop] → [Phase 4. Archive]
               ▲                                       ▲
           사람 판단 ①                              사람 판단 ②
           Spec 승인                             Epic 결과 확인
```

| Phase            | 입력                                                       | 출력                         | 스킬                 |
| ---------------- | ---------------------------------------------------------- | ---------------------------- | -------------------- |
| **1. Spec**      | AC 문서                                                    | tech-spec.md, review.md      | `/bf-spec`           |
| **2. Plan**      | tech-spec.md                                               | sprint-status.yaml, stories/ | `/bf-execute` (자동) |
| **3. Epic Loop** | stories/, sprint-status.yaml, conventions.md, tech-spec.md | 코드 커밋, review.md         | `/bf-execute` (자동) |
| **4. Archive**   | sprint-status.yaml, 산출물 전체                            | archive/, conventions.md     | `/bf-archive-sprint` |

Phase 3은 에픽 단위로 3개 Sub-Phase를 자율 실행한다:

| Sub-Phase          | 역할                                       | 상태 필드 (복구 기준)       |
| ------------------ | ------------------------------------------ | --------------------------- |
| **3a. Implement**  | Story별 TDD (Ralph Loop)                   | `story.status`, `story.tdd` |
| **3b. E2E Verify** | E2E 테스트, 실패 시 회귀 루프              | `epic.e2e`                  |
| **3c. Review**     | Epic 통합 리뷰 (Convention Guard + 다관점) | `story.review`              |

> 상세 흐름도 및 프로토콜: [BF-WORKFLOW-GRAPH.md](./BF-WORKFLOW-GRAPH.md)

## 사용자 스킬

### BF Workflow

| 스킬                     | 용도                                     |
| ------------------------ | ---------------------------------------- |
| `/bf-spec`               | AC 문서 → Tech Spec 작성 + 자동 리뷰     |
| `/bf-execute`            | Plan → Epic Loop 실행 (사람-시스템 경계) |
| `/bf-resume`             | 중단된 워크플로우 복구                   |
| `/bf-archive-sprint`     | 스프린트 아카이빙                        |
| `/bf-metrics`            | 메트릭 분석 및 최적화 제안 (선택)        |
| `/bf-update-conventions` | 컨벤션 규칙 업데이트                     |

### 유틸리티

| 스킬  | 용도                                                    |
| ----- | ------------------------------------------------------- |
| `/d`  | 영한 개발자 사전. 한국어 뜻 + 개발 맥락 (haiku 모델)   |

## Prerequisites

- **yq v4+** (Mike Farah's Go-based): sprint-status.yaml 프로그래밍적 업데이트에 사용
  ```bash
  # macOS
  brew install yq
  # Linux: https://github.com/mikefarah/yq#install
  ```

## 설치

```bash
# 전체 설치
npx skills add khaki4/my_skills -a claude-code

# 특정 스킬만 설치
npx skills add khaki4/my_skills --skill bf-spec -a claude-code

# 글로벌 설치 (모든 프로젝트에서 사용)
npx skills add khaki4/my_skills -a claude-code -g
```

## Quick Start

```bash
# 1. 스킬 설치
npx skills add khaki4/my_skills -a claude-code

# 2. 프로젝트 디렉토리에서 Claude Code 실행
cd your-project

# 3. AC 문서로 Tech Spec 작성 + 자동 리뷰
/bf-spec

# 4. Tech Spec 승인 후 구현 시작 (Plan → Epic Loop 자동 진행)
/bf-execute

# 5. 완료 후 아카이빙
/bf-archive-sprint
```

사람 판단은 2회만 요구된다: **Spec 승인** (`/bf-spec` 후)과 **Epic 결과 확인** (`/bf-execute` 중 각 에픽 완료 시).

## 설계 원칙

1. **상위 에이전트는 중간 과정을 모른다** — 컨텍스트 격리
2. **"done" + 파일만 전달** — 컨텍스트 소멸, 파일만 남는다
3. **사람 판단은 2개만** — Spec 승인 + Epic 결과 확인
4. **파일이 모든 맥락을 운반** — 대화를 통한 전달 없음
5. **Lead는 코드를 만지지 않음** — 모든 구현은 agent에게 위임

> 전체 지침: [CLAUDE.md](./CLAUDE.md)
