# BF Workflow 전체 프로세스 그래프

## 설계 원칙

> Canonical source: [CLAUDE.md — 설계 원칙](./CLAUDE.md#설계-원칙)

---

## 추상 흐름 (Phase Overview)

각 Phase는 독립적으로 구현을 변경·발전시킬 수 있는 단위이다.
Phase 간 계약은 **입력 파일 → Phase 실행 → 출력 파일**이며, 내부 구현은 자유롭게 교체 가능하다.

```
 AC 문서
   │
   ▼
┌──────────┐   tech-spec.md    ┌──────────┐   sprint-status.yaml
│  Phase 1 │ + review.md       │  Phase 2 │ + stories/
│   Spec   │──────────────────→│   Plan   │──────────────────────┐
│          │                   │          │                      │
└──────────┘                   └──────────┘                      │
      ▲                                                         │
      │ 수정요청                                                 ▼
 ─ ─ ─ ─ ─ ─ ─ ─                              ┌─────────────────────────────┐
   사람 판단 ①                                  │     Phase 3  Epic Loop      │
   Spec 승인                                   │        (×에픽 수)            │
                                               │                             │
                                               │  ┌─────────┐               │
                                               │  │ 3a. TDD  │  코드 + 커밋  │
                                               │  │Implement │──────┐       │
                                               │  └─────────┘      │       │
                                               │                    ▼       │
                                               │  ┌─────────┐  회귀 시     │
                                               │  │ 3b. E2E  │──┐ 3a 재실행 │
                                               │  │ Verify   │  │          │
                                               │  └────┬────┘  │          │
                                               │       │ pass   │          │
                                               │       ▼        │          │
                                               │  ┌─────────┐  │          │
                                               │  │3c.Review │  │          │
                                               │  │ Inspect  │──┘          │
                                               │  └────┬────┘             │
                                               │       │                   │
                                               └───────┼───────────────────┘
                                                       │
                                                       ▼
                                                  사람 판단 ②
                                               Epic 결과 확인
                                          (진행 / 수정 재실행 / 중단)
                                                       │
                                            ┌──────────┼──────────┐
                                            │ 다음 에픽 │  수정     │ 중단
                                            │ ↑ Loop   │ → 같은   │
                                            │          │   에픽    │
                                            │          │   재실행  │
                                            ▼          ▼          ▼
                                       ┌──────────┐          워크플로우
                                       │  Phase 4 │            종료
                                       │ Archive  │
                                       │ +Metrics │
                                       │ +Rules   │
                                       └──────────┘
```

### Phase 계약 요약

| Phase | 입력 | 출력 | 사람 판단 |
|-------|------|------|----------|
| **1. Spec** | AC 문서 | tech-spec.md, review.md | ① 승인/수정 |
| **2. Plan** | tech-spec.md | sprint-status.yaml, stories/ | — |
| **3. Epic Loop** | stories/, sprint-status.yaml, conventions.md, tech-spec.md | 코드 커밋, review.md, sprint-status.yaml | ② 진행/수정/중단 |
| **4. Archive** | sprint-status.yaml, 산출물 전체 | archive/, conventions.md, metrics | — |

### Phase 3 Sub-Phase 계약

Epic Loop 내부의 각 step은 독립된 입력-출력 계약을 가진다.
교체 시 해당 Sub-Phase만 변경하면 되고, 복구 시 상태 필드로 재진입 지점을 판단한다.

| Sub-Phase | 입력 | 출력 | 상태 필드 (복구 기준) |
|-----------|------|------|--------------------|
| **3a. Implement** | stories/, conventions.md | 코드 커밋, sprint-status.yaml (story 상태) | `story.status`, `story.tdd` |
| **3b. E2E Verify** | 코드 커밋, story AC | E2E 스크립트, 실행 결과 | `epic.e2e` |
| **3c. Review** | 전체 diff, conventions.md, tech-spec.md | review.md | `story.review` |

> 각 Sub-Phase 내부의 에이전트 구성, 조율 패턴, 모델 배당은 아래 상세 흐름에서 다룬다.

---

## 전체 흐름 (상세)

```
사용자
 │
 │  /bf-spec {AC 문서}
 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Main Session                                                   │
│  역할: 위임만 한다. 오케스트레이션 하지 않는다.                  │
│  컨텍스트: 최소 (각 단계의 "done" + 산출물 경로만 보유)          │
│                                                                 │
│  ┌───────────────────────────────────────┐                      │
│  │ /bf-spec (직접 실행)                   │                      │
│  │ 입력: AC 문서                           │                      │
│  │ 출력: docs/tech-specs/{TICKET}.md      │                      │
│  │ → "done" + tech-spec.md                │                      │
│  └────────────────┬──────────────────────┘                      │
│                   │ auto                                        │
│                   ▼                                             │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ bf-lead-review  (스폰, tech-spec 모드)                 │      │
│  │ 조율 패턴: discourse                                   │      │
│  │ 초기 로딩: tech-spec.md, conventions.md                │      │
│  │                                                       │      │
│  │   Reviewer-A ◄──► Reviewer-B ◄──► Reviewer-C         │      │
│  │       │               │               │               │      │
│  │       └───── 직접 대화로 합의 시도 ─────┘               │      │
│  │                   │                                   │      │
│  │              미합의 시 Lead 중재                        │      │
│  │                   │                                   │      │
│  │              그래도 미합의 → 미합의로 기록               │      │
│  │                                                       │      │
│  │ → "done" + review.md (합의/미합의 분리)                │      │
│  │ → 종료 (컨텍스트 소멸)                                 │      │
│  └───────────────────────────────────────────────────────┘      │
│                   │                                             │
│                   ▼                                             │
│          ┌────────────────┐                                     │
│          │  사람 판단 ①    │                                     │
│          │  Spec 승인/수정  │                                     │
│          └────────┬───────┘                                     │
│                   │ 승인                                        │
│                   ▼                                             │
│  /bf-execute ─→ 에픽 단위 루프 시작 (아래 참조)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## bf-execute — 사람-시스템 경계 허브

```
═══════════════════════════════════════════════════════════════════
  bf-execute (메인 세션)
  역할: 사람과 시스템 사이의 유일한 경계
  orchestrate를 모드별로 스폰, 에픽 결과를 사람에게 제시
═══════════════════════════════════════════════════════════════════
 │
 │  Step 2. Plan 단계
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ orchestrate (plan 모드) 스폰                                     │
│ → bf-lead-plan 스폰 → Story 구조 생성                            │
│ → "done" + sprint-status.yaml + stories/                        │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ▼
        에픽/스토리 구조를 사람에게 제시
                 │
                 ▼
  Step 3. 에픽 루프 시작 (에픽 순서대로)
```

---

## 에픽 루프 (bf-execute가 순회)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Step 3. 에픽 루프 (bf-execute가 순회)                          │
│                                                                 │
│   ┌─── Epic ────────────────────────────────────────────┐       │
│   │                                                     │       │
│   │  3a. orchestrate (epic 모드) 스폰                    │       │
│   │  ┌──────────────────────────────────────────────────┐│       │
│   │  │ orchestrate — 완전 자율 실행                       ││       │
│   │  │ 사람과 소통 없음                                   ││       │
│   │  │                                                  ││       │
│   │  │  E1. bf-lead-implement (스폰)                     ││       │
│   │  │  ┌────────────────────────────────────────────┐  ││       │
│   │  │  │ ※ 모든 Story를 agent에게 위임               │  ││       │
│   │  │  │   Lead는 코드를 직접 만지지 않음              │  ││       │
│   │  │  │                                            │  ││       │
│   │  │  │  S/M: 단독 Sonnet, ralph-loop              │  ││       │
│   │  │  │  L/XL: Lead(Opus) + Implementers(Sonnet)   │  ││       │
│   │  │  │                                            │  ││       │
│   │  │  │ → "done" / "done (stuck: ...)"             │  ││       │
│   │  │  └────────────────────────────────────────────┘  ││       │
│   │  │       │                                          ││       │
│   │  │       ▼  자동 판단 (stuck)                        ││       │
│   │  │  ├─ stuck Story → auto-skip                      ││       │
│   │  │  └─ 나머지로 E2E 진행                             ││       │
│   │  │       │                                          ││       │
│   │  │       ▼                                          ││       │
│   │  │  E2. E2E agent (스폰)                             ││       │
│   │  │  ┌────────────────────────────────────────────┐  ││       │
│   │  │  │ E2E 작성 → 즉시 실행                        │  ││       │
│   │  │  │ → "passed" / "failed" / "escalation"       │  ││       │
│   │  │  └────────────────────────────────────────────┘  ││       │
│   │  │       │                                          ││       │
│   │  │       ▼  자동 판단 (E2E)                          ││       │
│   │  │  ├─ passed → E3 리뷰                             ││       │
│   │  │  ├─ failed (사이클 < 2) → regression → E1 재실행  ││       │
│   │  │  ├─ failed (사이클 = 2) → max-regression-cycles   ││       │
│   │  │  └─ escalation → escalated 기록                   ││       │
│   │  │       │                                          ││       │
│   │  │       ▼                                          ││       │
│   │  │  E3. bf-lead-review (스폰, epic-review 모드)      ││       │
│   │  │  ┌────────────────────────────────────────────┐  ││       │
│   │  │  │ 리뷰어 discourse → review.md 생성            │  ││       │
│   │  │  │ 사람과 소통 없음 (자기 완결)                   │  ││       │
│   │  │  │ → "done: approved" / "done: blockers"       │  ││       │
│   │  │  └────────────────────────────────────────────┘  ││       │
│   │  │       │                                          ││       │
│   │  │       ▼  자동 판단 (리뷰)                         ││       │
│   │  │  ├─ approved → 에픽 완료                          ││       │
│   │  │  └─ blockers → 기록만, 자동 수정 안 함             ││       │
│   │  │                                                  ││       │
│   │  │ → "done" + sprint-status.yaml + review.md        ││       │
│   │  └──────────────────────────────────────────────────┘│       │
│   │       │                                              │       │
│   │       ▼                                              │       │
│   │  3b. 에픽 결과 제시 (bf-execute → 사람)               │       │
│   │  ┌──────────────────────────────────────────────────┐│       │
│   │  │ Story 상태 요약 (done/skipped/stuck)              ││       │
│   │  │ E2E 결과                                         ││       │
│   │  │ 리뷰 결과 (Blocker/Recommended 건수)              ││       │
│   │  │ stuck Story 정보 (있으면)                          ││       │
│   │  └──────────────────────────────────────────────────┘│       │
│   │       │                                              │       │
│   │       ▼                                              │       │
│   │  3c. 사람 판단                                        │       │
│   │  ├─ 진행 → 다음 에픽                                  │       │
│   │  ├─ 수정 → modification.md 생성 → 같은 에픽 재실행    │       │
│   │  └─ 중단 → 워크플로우 종료                             │       │
│   │                                                     │       │
│   └─────────────────────────────────────────────────────┘       │
│                     │                                           │
│                     ▼ 다음 에픽 있으면 Epic 루프 반복             │
│                     │ 없으면 ↓                                   │
│                                                                 │
│ Step 4. 전체 완료 → 후처리 안내                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## bf-lead-orchestrate — 모드 시스템

```
═══════════════════════════════════════════════════════════════════
  bf-lead-orchestrate
  핵심: 사람과 절대 소통하지 않는다
  모드 기반 자율 실행 → 결과 파일과 함께 종료
═══════════════════════════════════════════════════════════════════

  Plan 모드                         Epic 모드
  ┌─────────────┐                   ┌─────────────────────┐
  │ bf-lead-plan │                   │ E1: implement       │
  │ 스폰 → 수신  │                   │ → 자동 stuck 판단    │
  │ "done"       │                   │ E2: E2E             │
  └──────┬──────┘                   │ → 자동 E2E 판단      │
         │                          │ E3: review           │
         ▼                          │ → 자동 리뷰 판단     │
  "done" + stories/                 └──────┬──────────────┘
  + sprint-status.yaml                      │
                                            ▼
                                    "done" + review.md
                                    + sprint-status.yaml
```

---

## 후처리 (Main Session 복귀)

```
                      │
                      ▼  Main Session 복귀
             ┌────────────────────┐
             │ /bf-archive-sprint │  (Main Session 직접 실행)
             │ → archive/         │
             └────────┬───────────┘
                      │
                      ▼
             ┌────────────────────┐
             │ /bf-metrics        │  (선택, Main Session 직접)
             │ → dialog only      │
             └────────┬───────────┘
                      │
                      ▼
             ┌────────────────────────┐
             │ /bf-update-conventions  │  (Main Session 직접)
             │ → conventions.md        │
             └─────────────────────────┘
```

---

## Orchestrate 자동 판단 정책 요약

orchestrate는 분석하지 않는다. 자동 정책에 따라 분기한다.

### Stuck 자동 판단

| 조건 | 자동 결정 |
|------|----------|
| stuck 없음 | E2E로 진행 |
| stuck Story + 비stuck Story 존재 | stuck Story skip, 나머지로 E2E |
| 전 Story stuck | 모두 skip, E2E 진행 |

### E2E 자동 판단

| 조건 | 자동 결정 |
|------|----------|
| passed | review로 진행 |
| failed (E2E 사이클 < 2) | regression story → implement 재실행 |
| failed (E2E 사이클 = 2) | `e2e: max-regression-cycles` 기록 → review |
| escalation (가드레일 초과 또는 인프라 오류) | `e2e: escalated` 기록 → review |

### Review 자동 판단

| 조건 | 자동 결정 |
|------|----------|
| Blocker 0건 (approved) | 에픽 완료 |
| Blocker 1건+ (blockers) | 기록만, 자동 수정 안 함 → 에픽 완료 |

Blocker가 있어도 자동 수정하지 않는다. 사람이 bf-execute의 에픽 결과에서 판단한다.

---

## 쟁점 해소 프로토콜 (모든 Lead에 공통 적용)

```
  Teammate-A ◄────────► Teammate-B
       │     직접 대화     │
       └────────┬─────────┘
                │
           합의됨? ── Yes ──→ Lead에 결론만 보고
                │
               No
                │
                ▼
         Lead가 중재
     (프로젝트 방향성 기준)
                │
           합의됨? ── Yes ──→ Lead 결정으로 확정
                │
               No
                │
                ▼
        미합의로 기록 후 버림
     (최종 결과에 "미합의 쟁점"으로 포함)
     (Lead의 최종 판단 + 근거 반드시 포함)
```

---

## 에스컬레이션 프로토콜

"stuck"은 중간 과정이 아니라 최종 상태이다. 각 층은 자기 범위에서 판단하고, orchestrate가 자동 skip한다. 사람은 bf-execute의 에픽 결과에서 확인한다.

```
story agent
  → "stuck" + stuck.md (시도한 접근들, 에러 내용 기록)
  → 종료

bf-lead-implement
  → 다른 Story들은 계속 진행
  → 모든 Story 완료 후 orchestrate에 보고:
    "done (stuck: Story-N)" + sprint-status.yaml + stuck.md
  → 종료

orchestrate (자동 판단)
  → stuck Story → auto-skip (status: skipped)
  → 나머지 Story로 E2E → review 진행
  → 결과 파일과 함께 종료

bf-execute (사람 경계)
  → 에픽 결과에 skipped Story 표시
  → 사람이 수정 재실행 선택 시 modification.md로 지시
```

---

## 수정 지시 전달 프로토콜

수정 지시는 사람 → bf-execute → modification.md → orchestrate 경로로 전달된다.

```
사람: "Story-2 에러 핸들링 바꿔"
  → bf-execute가 modification.md에 수정 지시 원문 기록
    → orchestrate (epic 모드, modification_path 전달)
      → modification.md 읽어 수정 대상 Story 파악
        → bf-lead-implement: 해당 Story만 재구현
```

modification.md 형식:

```markdown
# {EPIC-ID} Modification

## 수정 지시
{사람이 입력한 수정 내용 원문}

## 대상 Story
- {Story ID 목록}
```

모든 단계가 같은 파일을 읽는다. 요약이 끼어들 여지가 없다.

---

## sprint-status.yaml 쓰기 권한

> Canonical source: [CLAUDE.md — sprint-status.yaml 쓰기 권한](./CLAUDE.md#sprint-statusyaml-쓰기-권한)

---

## 모델 배당

orchestrate와 lead-plan은 전체 구조를 결정하므로 항상 Opus.
lead-implement와 lead-review는 에픽 구성에 따라 모델이 달라진다.

| Lead | Opus 조건 | Sonnet 조건 |
|------|-----------|------------|
| bf-lead-orchestrate | 항상 | — |
| bf-lead-plan | 항상 | — |
| bf-lead-implement | 에픽에 L/XL Story 포함 | S/M만 |
| bf-lead-review | 에픽에 L/XL Story 포함 | S/M만 |

orchestrate가 sprint-status.yaml의 난이도 태그를 읽고 모델을 결정한다.

Story agent 모델:

| 난이도 | 위임 방식 | 모델 |
|--------|----------|------|
| S | 단독 agent 1개 | Sonnet |
| M | 단독 agent 1개 | Sonnet |
| L | Lead(Opus) + Impl teammates | Opus + Sonnet |
| XL | Lead(Opus) + 3+ teammates | Opus + Sonnet/Opus |

Review agent 모델:

| 모드 | 리뷰어 모델 |
|------|------------|
| tech-spec | 전원 Opus |
| epic-review | Lead와 동일 (L/XL 포함 시 Opus, S/M만이면 Sonnet) |

---

## 컨텍스트 수명

```
  Main Session ·······························→ (세션 끝까지 유지)
       │                                        컨텍스트: 최소
       │
       ├─ Orchestrate (plan) ──┤ 종료           에픽/Story 구조 생성
       │       │
       │       └─ Lead-Plan ───┤ 종료
       │             ├─ Creator-A ┤ 종료
       │             └─ Creator-B ┤ 종료
       │
       │  에픽 루프 (bf-execute가 순회):
       │
       ├─ Orchestrate (epic-1) ┤ 종료           에픽당 1회 스폰
       │       │
       │       ├─ Lead-Impl ───┤ 종료
       │       │     ├─ Agent-S ──┤ 종료
       │       │     ├─ Agent-M ──┤ 종료
       │       │     └─ Impl-A ───┤ 종료
       │       │
       │       ├─ E2E Agent ───┤ 종료
       │       │
       │       └─ Lead-Review ─┤ 종료           자기 완결 (사람 소통 없음)
       │             ├─ Rev-A ───┤ 종료
       │             └─ Rev-B ───┤ 종료
       │
       │  ← 에픽 결과 확인 (사람 판단) →
       │
       ├─ Orchestrate (epic-2) ┤ 종료           다음 에픽
       │       │ ...
       │
       ├─ archive (직접) ─┤
       ├─ metrics (직접) ─┤
       └─ conventions (직접) ─┤

  ────────────────────────────────────────→ 시간
  │← 파일만 남는다 →│
```

---

## 사람 판단 요약

| 판단 | 시점 | 내용 |
|------|------|------|
| ① Spec 승인 | Tech Spec 리뷰 후 (메인 세션) | 승인 / 수정요청 |
| Epic 결과 확인 | 각 에픽 완료 후 (bf-execute) | 진행 / 수정 재실행 / 중단 |

**사람 판단은 2종류만 존재한다.** 시스템 내부 에이전트는 사람과 절대 직접 소통하지 않는다.

**"진행" 선택 시 상태 정리:** 사람이 "다음 에픽으로 진행"을 선택하면, bf-execute/bf-resume가 해당 에픽의 sprint-status.yaml을 정리한다: skipped Story의 `review: approved`, Blocker 수용 시 done Story의 `review: approved`. 이는 archive 전제조건 충족을 보장한다.

전체 워크플로우 사람 판단 횟수: **① + 에픽 수**

---

## Lead 스킬 요약

| Lead | 조율 패턴 | 초기 로딩 | 역할 | 모델 |
|------|-----------|----------|------|------|
| bf-lead-orchestrate | sequence | tech-spec, sprint-status | 모드 기반 자율 실행 (plan/epic). 사람 소통 없음 | 항상 Opus |
| bf-lead-plan | distribute | tech-spec, conventions | 에픽/스토리 구조 결정, 병렬 분배 + 취합 | 항상 Opus |
| bf-lead-implement | monitor | story 문서, conventions, sprint-status, modification.md(수정 시) | agent 스폰 + "done"/"stuck" 수신 + sprint-status 업데이트 | Opus/Sonnet |
| bf-lead-review | discourse | tech-spec, conventions, 전체 diff | 자기 완결적 review.md 생성. 사람 소통 없음 | Opus/Sonnet |
