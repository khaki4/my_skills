# CLAUDE.md

이 파일은 Claude Code (claude.ai/code)가 이 저장소의 코드를 다룰 때 참고하는 지침이다.

## 저장소 개요

이 저장소는 **BF (Brownfield) Workflow**를 구현하는 Claude Code 커스텀 스킬 모음이다. 브라운필드 프로젝트를 위한 구조화된 개발 파이프라인으로, **4개 Phase**로 구성되며 각 Phase는 독립적인 입력-출력 계약을 가져 내부 구현을 다른 Phase에 영향 없이 변경할 수 있다:

| Phase | 요약 |
|-------|------|
| **1. Spec** | AC 문서 → Tech Spec + 다관점 리뷰 |
| **2. Plan** | Tech Spec → Epic/Story 분해 |
| **3. Epic Loop** | TDD 구현 → E2E 검증 → 통합 리뷰 (에픽 단위) |
| **4. Archive** | 스프린트 아카이빙 + 메트릭 + 컨벤션 규칙 |

> Phase 상세 및 Sub-Phase 계약: **[BF-WORKFLOW-GRAPH.md](./BF-WORKFLOW-GRAPH.md)**

## 설계 원칙

1. **상위 에이전트는 중간 과정을 모른다** — 컨텍스트 격리, 오염 방지
2. **상위 에이전트는 방향성을 지킨다** — 팀메이트 대화 정리, 프로젝트 방향 기준으로 조율/결정
3. **책임 단위 종료 시 "done" + 파일만 전달** — 컨텍스트는 소멸, 파일만 남는다
4. **쟁점 해소**: 팀메이트 직접 대화 → Lead 중재 → 미합의 시 버림(기록)
5. **파일이 모든 맥락을 운반한다** — 수정 지시, 리뷰 결과, stuck 보고 등 모든 전달은 파일 기반
6. **단계별 단일 쓰기 지점** — sprint-status.yaml 동시 쓰기 방지, Lead가 통합 업데이트
7. **사람 = 외부 경계의 느린 에이전트** — 사람은 bf-execute(메인 세션)를 통해서만 시스템과 소통. 판단 지점은 2개: Spec 승인 + Epic 결과 확인

## 스킬 아키텍처

모든 스킬은 `skills/{skill-name}/SKILL.md`에 위치한다. 일관된 구조를 따른다:
- YAML frontmatter (`name`, `description`)
- Overview, When to Use, Instructions, Output Format 섹션
- 스킬 간 연결은 Lead 계층을 통해 (플랫 체인 아님)

### Lead 스킬 계층

BF workflow는 4개 Lead 스킬을 사용하며, 각각 고유한 조율 패턴을 가진다:

| Lead | 조율 패턴 | 역할 | 모델 |
|------|----------|------|------|
| **bf-lead-orchestrate** | Sequence | 모드 기반 자율 실행 (plan/epic). 사람 소통 없음 | plan: Sonnet, epic: Opus |
| **bf-lead-plan** | Distribute | Epic/Story 구조, 병렬 분배 + 취합 | 항상 Opus |
| **bf-lead-implement** | Monitor | agent 스폰 + "done"/"stuck" 수신 + 상태 업데이트 | Opus/Sonnet |
| **bf-lead-review** | Discourse | 자기 완결적 review.md 생성. 사람 소통 없음 | Opus/Sonnet |

- **bf-lead-orchestrate**: plan 모드는 bf-lead-plan 스폰 후 결과를 전달하는 단순 라우터 역할이므로 Sonnet. epic 모드는 자동 판단 정책(stuck/E2E/review)을 실행하므로 Opus
- **bf-lead-implement/bf-lead-review**: 에픽에 L/XL Story가 포함되면 Opus, S/M만이면 Sonnet

### 워크플로우 시퀀스

> Phase Overview, Sub-Phase 계약, ASCII 다이어그램, 프로토콜 상세: **[BF-WORKFLOW-GRAPH.md](./BF-WORKFLOW-GRAPH.md)**

BF workflow는 다음 순서로 실행된다:

**Phase 1. Spec**
1. **`/bf-spec`** → 사람이 AC 문서 제공 → Tech Spec 작성
2. **`bf-lead-review`** (자동, tech-spec 모드) → Agent Teams 다관점 리뷰
3. **사람 판단 ①: Spec 승인** → 승인 또는 수정 요청

**Phase 2. Plan**
4. **`/bf-execute`** → **`bf-lead-orchestrate`** (plan 모드) → Epic/Story 구조 생성

**Phase 3. Epic Loop** (에픽 단위, 순차)
5. 각 에픽은 **`bf-lead-orchestrate`** (epic 모드)를 통해 3개 Sub-Phase를 자율 실행:
   - **3a. Implement** → Story별 TDD (Ralph Loop)
   - **3b. E2E Verify** → E2E 테스트 실행, 실패 시 회귀 루프
   - **3c. Review** → Epic 통합 리뷰 (Convention Guard + 다관점)
6. **사람 판단 ②: Epic 결과 확인** → 진행 / 수정 재실행 (modification.md) / 중단

**Phase 4. Archive**
7. **`/bf-archive-sprint`** → 산출물을 archive로 이동, changelog 업데이트
8. **`/bf-metrics`** → 스프린트 메트릭 분석 및 최적화 제안 (선택)
9. **`/bf-update-conventions`** → 패턴 추출 및 컨벤션 규칙 업데이트

- **`/bf-resume`** → 중단된 워크플로우 복구 (수동, bf-execute와 동일한 에픽 루프)

### 조율 패턴 (Canonical Definition)

BF Lead 스킬과 범용 `/teams` 스킬이 공유하는 4가지 조율 패턴이다. `/teams` 스킬은 이 정의를 참조한다.

| 패턴 | 핵심 동작 | BF 사용처 | /teams 사용 |
|------|----------|----------|------------|
| **Distribute** | Lead가 작업 분할 → agent 병렬 실행 → Lead 취합 | bf-lead-plan | 병렬 구현, 리서치 |
| **Monitor** | Lead가 agent 스폰 → "done"/"stuck" 모니터링 → 상태 업데이트 | bf-lead-implement | TDD 구현, 장시간 작업 |
| **Discourse** | 독립 분석 → 교차 검증 → 합의/미합의 분리 | bf-lead-review | 코드 리뷰, 설계 의사결정 |
| **Sequence** | 단계별 트리거, 분기만, 분석 없음 | bf-lead-orchestrate | 멀티 페이즈 파이프라인 |

## 핵심 개념

### Ralph Loop

**Ralph Loop**은 단일 story agent가 외부 조율 없이 실행하는 반복적 TDD 사이클이다:
1. AC 기반 테스트 작성
2. 테스트 실행 → Red 확인 (실패)
3. 테스트 통과를 위한 코드 구현
4. 테스트 실행 → Green 확인 (성공) — **최대 5회 재시도, 동일 에러 2회 연속 시 접근 전환**
5. 필요 시 리팩토링
6. 변경사항 커밋

모든 난이도에 사용 — 차이는 agent 구성 (S/M: 단독 Sonnet, L: Opus lead + Sonnet implementers, XL: Opus lead + 3+ teammates). Ralph Loop 가드레일은 `skills/bf-lead-implement/SKILL.md`에 인라인 정의.

### 난이도별 실행 전략

각 Story는 agent 구성을 결정하는 난이도 태그가 부여된다:

- **S (Simple)**: 단일 파일, 명확한 AC, 의존성 없음
  - 실행: 단독 Sonnet agent
  - 리뷰: Epic 통합 리뷰

- **M (Medium)**: 2-3개 파일, 모듈 간 연결
  - 실행: 단독 Sonnet agent
  - 리뷰: Epic 통합 리뷰

- **L (Large)**: 다수 파일, 아키텍처 영향 큼
  - 실행: Sub-Lead (Opus) + Sonnet implementers
  - 리뷰: discourse 포함 Epic 통합 리뷰

- **XL (Complex)**: 크로스 레이어, 보안/성능 고려, 설계 결정
  - 실행: Sub-Lead (Opus) + 3+ teammates
  - 리뷰: 확장 discourse 포함 Epic 통합 리뷰

**중요**: Story 단위 리뷰는 없다. 모든 리뷰는 E2E 통과 후 Epic 단위로 수행.

### Agent Teams 패턴

- **bf-execute가 사람 경계**: bf-execute만 사람과 소통. 내부 agent는 사람과 직접 소통 없음
- **Epic 실행은 순차**: 에픽은 의존성 순서대로 하나씩. 향후 에픽 간 의존성이 없는 경우 병렬 실행 가능성을 검토할 수 있으나, 현재는 sprint-status.yaml 단일 쓰기 원칙과 복잡도 관리를 위해 순차 실행
- **Story 실행은 병렬**: 에픽 내 Story는 파일 겹침이 없으면 동시 실행 가능. **단, lock 파일(package-lock.json 등), 공유 설정 파일, 자동 생성 파일은 "겹침"으로 간주**하여 관련 Story를 순차 실행 (상세: `bf-lead-implement/SKILL.md` 병렬 실행 규칙)
- **Lead는 코드를 직접 만지지 않음**: bf-lead-implement는 모든 Story를 agent에게 위임
- **sprint-status.yaml 단일 쓰기**: 현재 단계 담당 Lead만 쓰기
- **orchestrate는 에픽 단위**: 에픽 당 1회 스폰, 결과 파일과 함께 종료

### 파일 구조 규칙

이 스킬을 사용하면 대상 프로젝트에 다음 디렉토리 구조가 생성된다:

```
docs/
  tech-specs/
    {TICKET}-tech-spec.md
  stories/
    {TICKET}-story-{N}.md
  reviews/
    {TICKET}-tech-spec-review.md
    {EPIC-ID}-review.md
    {EPIC-ID}-modification.md    # 사람 수정 지시 (재실행 시)
  sprint-status.yaml
  conventions.md          # Convention Guard 규칙 (/bf-spec이 초기 seed 생성)
  #   섹션: Core (항상 전달) = Architecture, Naming, Testing, Code Style
  #         Concern-area (기술 스택별) = UI/API/Database/Security/Infrastructure Patterns
  #   bf-lead-implement가 Story 파일 경로 기반으로 관련 섹션만 필터링하여 agent에 인라인 전달
  #   bf-lead-implement가 Story의 "주요 라이브러리"를 context7로 조회하여 library reference도 함께 인라인 전달 (optional)
  archive/
    {SPRINT-XX}/
      tech-specs/
      stories/
      reviews/
      sprint-status.yaml

.ralph-progress/          # Ralph Loop 크래시 복구용 (임시, .gitignore 권장)
  {STORY-ID}.json

docs/
  reviews/
    {STORY-ID}-stuck.md     # Ralph Loop stuck 보고서 (bf-lead-implement가 저장)

tests/
  e2e/
    {epic-name}/
      {scenario-name}.sh
```

### sprint-status.yaml 구조

모든 Epic/Story 진행 상황을 추적한다:

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

상태 필드 값:
- **status**: `todo` → `in_progress` → `done` (stuck Story를 orchestrate가 자동 skip 시 `skipped`). 수정 재실행(modification) 시 `done` → `in_progress` 역전이 허용
- **tdd**: `pending` → `done`
- **review**: `pending` → `approved`
- **e2e**: `pending` → `passed` | `skipped` | `escalated` | `max-regression-cycles` (terminal states: passed/skipped/escalated/max-regression-cycles)

메트릭 필드 값 (하위 agent가 기록, 기본값으로 초기화):
- **model_used**: `null` → `"sonnet"` | `"opus-lead"` | `"opus-lead+3"` (bf-lead-implement 기록)
- **ralph_retries**: `0` → Green 검증 실패 재시도 횟수 (bf-lead-implement 기록)
- **ralph_approaches**: `0` → Stuck Detection 접근 전환 횟수 (bf-lead-implement 기록)
- **review_blockers**: `0` → Blocker 수 (bf-lead-review 기록)
- **review_recommended**: `0` → Recommended 수 (bf-lead-review 기록)
- **failure_tag**: `null` → 실패 태그 (E2E agent가 regression story에만 기록)
- **is_regression**: `false` → E2E 실패에서 자동 생성된 Story 여부 (E2E agent 기록)
- **parent_story**: `null` → regression 원본 Story ID (E2E agent 기록)
- **ralph_stuck**: `false` → Ralph Loop 한도 초과 시 `true` (bf-lead-implement 기록)

### sprint-status.yaml 쓰기 권한

각 단계가 순차적이므로 동시 쓰기가 발생하지 않는다. Story agent는 sprint-status.yaml을 만지지 않는다.

| 에이전트 | 권한 | 쓰기 내용 |
|----------|------|----------|
| Story agent | 없음 (코드+커밋만) | — |
| bf-lead-implement | 쓰기 | Story 상태, 메트릭 (retries, approaches, stuck) |
| E2E agent | 쓰기 | `e2e: passed`, failure tag, regression story 추가 |
| bf-lead-review | 쓰기 | review 상태, blocker/recommended 수 |
| bf-lead-orchestrate | 페이즈 전환 쓰기 | status 전환 (in_progress, skipped), `e2e: escalated`/`max-regression-cycles`, orphan regression story skip |
| bf-execute / bf-resume | 사람 판단 결과 쓰기 | 사람이 "진행" 선택 시: skipped Story `review: approved`, Blocker 수용 시 done Story `review: approved` |

### sprint-status.yaml 업데이트 프로토콜

sprint-status.yaml 업데이트 시 **Read-yq-Verify** 프로토콜을 따른다:

1. **Read**: 수정 전 sprint-status.yaml을 읽어 현재 상태 확인
2. **Update**: `yq -i` 명령으로 프로그래밍적 YAML 필드 업데이트
3. **Verify**: 업데이트 후 파일을 다시 읽어 변경 확인
4. **Recover** (Verify 실패 시): `git checkout docs/sprint-status.yaml`로 마지막 커밋 상태로 복원 → 1회 재시도. 재시도도 실패하면 stuck 보고

**핵심 규칙:**
- **최소 범위**: 자신에게 할당된 Story/Epic 필드만 변경. 다른 Story는 절대 수정하지 않음

**yq 전제조건**: `yq` (Mike Farah's Go-based, v4+)가 설치되어 있어야 한다. 각 Lead/E2E agent가 시작 시 1회 확인:
```bash
command -v yq >/dev/null 2>&1 || { echo "❌ yq not installed. Install and retry:"; echo "  macOS: brew install yq"; echo "  Linux: https://github.com/mikefarah/yq#install"; exit 1; }
```

**yq Fallback 전략**: yq를 설치할 수 없는 환경에서는 다음 대안을 순서대로 시도:
1. **Python fallback**: `python3 -c "import yaml; ..."` 로 YAML 업데이트 (PyYAML 필요)
2. **Edit tool fallback**: Claude Code의 Edit tool로 sprint-status.yaml을 직접 편집. 이 경우 Verify 단계에서 Read tool로 변경을 확인
3. Fallback 사용 시 결과에 "yq unavailable, used {fallback method}" 명시

**yq 사용 예시:**
```bash
# Story 상태 업데이트
yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "done"' docs/sprint-status.yaml

# 여러 필드 동시 업데이트
yq -i '.<SPRINT>.<EPIC>.<STORY>.status = "done" | .<SPRINT>.<EPIC>.<STORY>.tdd = "done"' docs/sprint-status.yaml

# regression story 추가
yq -i '.<SPRINT>.<EPIC>.<NEW-STORY> = {"status":"todo","difficulty":"S","tdd":"pending","review":"pending","model_used":null,"ralph_retries":0,"ralph_approaches":0,"review_blockers":0,"review_recommended":0,"failure_tag":"impl-bug","is_regression":true,"parent_story":"story-1","ralph_stuck":false}' docs/sprint-status.yaml
```

### TDD 사이클 구현

모든 Story는 엄격한 TDD (Red-Green-Refactor)를 따른다. 최대 5회 재시도, 동일 에러 2회 연속 시 접근 전환 (stuck detection). 크래시 복구를 위해 `.ralph-progress/{STORY-ID}.json`에 진행 상태를 기록한다.

> 상세 Ralph Loop 지침 및 가드레일: `skills/bf-lead-implement/SKILL.md` — "Story Agent용 Ralph Loop 지침"

### Epic 통합 리뷰 (Open Code Review + Convention Guard)

리뷰는 E2E 통과 후 **Epic 단위**로 수행된다 (Story 단위 아님). Convention Guard (필수) + 추가 리뷰어 (1-2명)가 Discourse 패턴으로 독립 분석 → 교차 검증 → 합의/미합의 분리. 자기 완결적 review.md를 생성하며 사람과의 실시간 Q&A 없음.

> 상세 리뷰 프로세스 및 모드별 동작: `skills/bf-lead-review/SKILL.md`

### 쟁점 해소 프로토콜

모든 Lead 스킬과 `/teams` 스킬에서 팀메이트 간 의견 충돌 시 적용:
1. **팀메이트 직접 대화**: SendMessage로 직접 반론/동의/보충 → 합의 결과를 Lead에 보고
2. **미합의 시 Lead 중재**: Lead가 프로젝트 방향(tech-spec, conventions) 기준으로 결정
3. **그래도 미합의 → 버림(기록)**: "미합의 쟁점"으로 결과에 포함, Lead 최종 판단 + 근거 기재, 토큰 소비 중단

> 프로토콜 적용 상세: [BF-WORKFLOW-GRAPH.md — 쟁점 해소 프로토콜](./BF-WORKFLOW-GRAPH.md#쟁점-해소-프로토콜-모든-lead에-공통-적용)

### 에스컬레이션 프로토콜

"stuck"은 중간 과정이 아니라 최종 상태이다. `Story agent → stuck.md → 종료 → bf-lead-implement가 나머지 계속 → orchestrate가 auto-skip → bf-execute가 사람에게 제시`. 전 Story stuck 시 `e2e: skipped` 처리.

> 상세 에스컬레이션 체인: [BF-WORKFLOW-GRAPH.md — 에스컬레이션 프로토콜](./BF-WORKFLOW-GRAPH.md#에스컬레이션-프로토콜)

### 메트릭 및 최적화

`/bf-metrics`는 sprint-status.yaml의 메트릭 데이터를 분석하여 모델 배당 최적화 및 난이도 재태깅을 **제안**한다 (읽기 전용, 사람이 결정). 실행 순서: `/bf-archive-sprint` → `/bf-metrics` (선택) → `/bf-update-conventions`.

> 상세 분석 기준 및 임계값: `skills/bf-metrics/SKILL.md`

> 참고 기술 및 방법론: **[REFERENCES.md](./REFERENCES.md)**

## 새 스킬 추가

이 저장소에 새 스킬을 만들 때:

1. 디렉토리 생성: `skills/{skill-name}/`
2. YAML frontmatter가 포함된 `SKILL.md` 추가:
   ```yaml
   ---
   name: skill-name
   description: 스킬 탐색을 위한 간단한 설명
   ---
   ```
3. 다음 섹션 포함: Overview, When to Use, Instructions, Output Format
4. 스킬이 파이프라인에 통합되면 이 CLAUDE.md의 워크플로우 시퀀스 업데이트
5. `npx skills add {username}/my-skills --skill {skill-name} -a claude-code`로 테스트

## 주요 참고사항

- **사람 판단 지점은 정확히 2개**: ① Spec 승인 (메인 세션, /bf-spec 이후)과 Epic 결과 확인 (bf-execute, 각 에픽 완료 후). 시스템 내부에 다른 사람 상호작용 없음
- **컨텍스트 격리**: 메인 세션 → orchestrate → Leads → agents. 각 층은 "done" + 파일로 종료, 컨텍스트 소멸
- **파일이 모든 맥락을 운반**: 수정 지시는 modification.md, 리뷰 결과는 review.md, stuck 보고는 stuck.md — 대화를 통한 전달 없음
- **Lead는 코드를 만지지 않음**: bf-lead-implement는 모든 난이도의 Story를 agent에게 위임
- **bf-execute가 유일한 사람 경계**: 내부 agent(orchestrate, review, implement)는 사람과 직접 소통 없음
- **Append-only changelog**: 대상 프로젝트의 CLAUDE.md changelog는 스프린트 이력 추적을 위해 추가 전용
- **Convention 축적**: `docs/conventions.md`는 스프린트를 거듭하며 패턴이 발견·체계화되면서 성장
- **E2E용 Agent-browser**: 브라우저 UI 기반 E2E 테스트는 CSS 셀렉터가 아닌 accessibility tree의 @ref 기반 요소 선택 사용 (API-only/CLI 프로젝트에는 해당 없음)
