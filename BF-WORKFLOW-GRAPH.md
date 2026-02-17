# BF Workflow 전체 프로세스 그래프

## 설계 원칙

1. **상위 에이전트는 중간 과정을 모른다** — 컨텍스트 격리, 오염 방지
2. **상위 에이전트는 방향성을 지킨다** — 팀메이트 대화 정리, 프로젝트 방향 기준으로 조율/결정
3. **책임 단위 종료 시 "done" + 파일만 전달** — 컨텍스트는 소멸, 파일만 남는다
4. **쟁점 해소**: 팀메이트 직접 대화 → Lead 중재 → 미합의 시 버림(기록)
5. **파일이 모든 맥락을 운반한다** — 수정 지시, 리뷰 결과, stuck 보고 등 모든 전달은 파일 기반
6. **각 단계는 단일 쓰기 지점** — sprint-status.yaml 동시 쓰기 방지, Lead가 통합 업데이트

---

## 전체 흐름

```
사용자
 │
 │  /bf-spec {AC + Slack 컨텍스트}
 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Main Session                                                   │
│  역할: 위임만 한다. 오케스트레이션 하지 않는다.                  │
│  컨텍스트: 최소 (각 단계의 "done" + 산출물 경로만 보유)          │
│                                                                 │
│  ┌───────────────────────────────────────┐                      │
│  │ /bf-spec (직접 실행)                   │                      │
│  │ 입력: AC + Slack 논의                  │                      │
│  │ 출력: docs/tech-specs/{TICKET}.md      │                      │
│  │ → "done" + tech-spec.md                │                      │
│  └────────────────┬──────────────────────┘                      │
│                   │ auto                                        │
│                   ▼                                             │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ bf-lead-review  (스폰)                                 │      │
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
│          │  사람 개입 ①    │                                     │
│          │  승인 / 수정요청 │                                     │
│          └────────┬───────┘                                     │
│                   │ 승인                                        │
│                   ▼                                             │
│  /bf-execute ─→ bf-lead-orchestrate (스폰)                      │
│                                                                 │
│  ← "done" + sprint-status.yaml (전체 에픽 완료) 수신 대기       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## bf-lead-orchestrate

```
═══════════════════════════════════════════════════════════════════
  bf-lead-orchestrate
  조율 패턴: sequence
  초기 로딩: tech-spec.md, sprint-status.yaml
  역할: "done" 수신 → 다음 Lead 트리거. 중간 과정 모름.
        분석하지 않는다. 시퀀싱만 한다.
═══════════════════════════════════════════════════════════════════
 │
 │  1. 에픽/스토리 생성
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ bf-lead-plan  (스폰)                                            │
│ 조율 패턴: distribute (분배 + 취합)                              │
│ 초기 로딩: tech-spec.md, conventions.md                         │
│                                                                 │
│   Lead가 Tech Spec 분석 → 에픽/스토리 구조 결정                  │
│                                                                 │
│   Story >= 5개일 때:                                             │
│     Creator-A ──→ story-1.md    (병렬)                           │
│     Creator-B ──→ story-2.md    (병렬)                           │
│     Creator-C ──→ story-3.md    (병렬)                           │
│     Lead 취합 → sprint-status.yaml 생성                         │
│                                                                 │
│ → "done" + stories/, sprint-status.yaml                         │
│ → 종료 (컨텍스트 소멸)                                           │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │  사용자 확인     │
        │  난이도 태그 조정 │
        └────────┬───────┘
                 │
                 ▼
```

---

## 에픽 루프

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   2. 에픽 루프 (orchestrate가 순회, 에픽 순서대로)               │
│                                                                 │
│   ┌─── Epic ────────────────────────────────────────────┐       │
│   │                                                     │       │
│   │  ┌→ 2a. 스토리 구현                                  │       │
│   │  │  ┌──────────────────────────────────────────────┐│       │
│   │  │  │ bf-lead-implement  (스폰)                     ││       │
│   │  │  │ 조율 패턴: monitor (모니터링 + 에스컬레이션)   ││       │
│   │  │  │ 초기 로딩: story 문서, conventions.md          ││       │
│   │  │  │ 모델: 에픽 내 L/XL 있으면 Opus, 없으면 Sonnet ││       │
│   │  │  │                                              ││       │
│   │  │  │ ※ 모든 난이도의 Story를 agent에게 위임한다.   ││       │
│   │  │  │   Lead는 코드를 직접 만지지 않는다.            ││       │
│   │  │  │   sprint-status.yaml은 Lead만 쓴다.           ││       │
│   │  │  │                                              ││       │
│   │  │  │  ┌─ S/M Story ──────────────────────────┐    ││       │
│   │  │  │  │  단독 agent(Sonnet) 스폰               │    ││       │
│   │  │  │  │  ralph-loop 실행                       │    ││       │
│   │  │  │  │  Red → Green (최대 5회 재시도)          │    ││       │
│   │  │  │  │  동일 에러 2연속 → 접근 전환            │    ││       │
│   │  │  │  │  → "done" + commit                    │    ││       │
│   │  │  │  │  → 또는 "stuck" + stuck.md            │    ││       │
│   │  │  │  └────────────────────────────────────────┘    ││       │
│   │  │  │                                              ││       │
│   │  │  │  ┌─ L/XL Story ─────────────────────────┐    ││       │
│   │  │  │  │  Lead(Opus) + Implementers(Sonnet)    │    ││       │
│   │  │  │  │                                      │    ││       │
│   │  │  │  │  Impl-A ◄──► Impl-B                  │    ││       │
│   │  │  │  │     └── 직접 대화로 해결 시도 ──┘      │    ││       │
│   │  │  │  │            │                         │    ││       │
│   │  │  │  │       미합의 → Lead 중재              │    ││       │
│   │  │  │  │            │                         │    ││       │
│   │  │  │  │       그래도 미합의 → 버린다 (기록)    │    ││       │
│   │  │  │  │  → "done" + commit                    │    ││       │
│   │  │  │  │  → 또는 "stuck" + stuck.md            │    ││       │
│   │  │  │  └──────────────────────────────────────┘    ││       │
│   │  │  │                                              ││       │
│   │  │  │ Lead가 "done" 수신 시 sprint-status.yaml 업데이트│       │
│   │  │  │                                              ││       │
│   │  │  │ 완료 신호 (orchestrate에 전달):               ││       │
│   │  │  │ → "done" + sprint-status.yaml               ││       │
│   │  │  │ → 또는 "done (stuck: Story-N)" + stuck.md   ││       │
│   │  │  │ → 종료 (컨텍스트 소멸)                        ││       │
│   │  │  └──────────────────────────────────────────────┘│       │
│   │  │           │                                      │       │
│   │  │           ▼                                      │       │
│   │  │                                                  │       │
│   │  │  orchestrate 분기:                                │       │
│   │  │  ├─ stuck 있음 → 사람에게 판단 요청               │       │
│   │  │  │   사람 선택:                                   │       │
│   │  │  │   ├─ AC 수정 → 해당 Story 재시도 (2a로)        │       │
│   │  │  │   ├─ Story 스킵 → 에픽에서 제외하고 진행       │       │
│   │  │  │   └─ 에픽 재설계 → lead-plan으로               │       │
│   │  │  │                                               │       │
│   │  │  └─ stuck 없음 → 2b로 진행                        │       │
│   │  │                                                  │       │
│   │  │           ▼                                      │       │
│   │  │  2b. E2E 작성 + 실행                              │       │
│   │  │  ┌──────────────────────────────────────────┐    │       │
│   │  │  │ E2E agent  (스폰)                         │    │       │
│   │  │  │                                          │    │       │
│   │  │  │ 구현된 코드 기반으로 E2E 작성              │    │       │
│   │  │  │ → 즉시 실행                               │    │       │
│   │  │  │                                          │    │       │
│   │  │  │ 통과:                                     │    │       │
│   │  │  │ → "passed" + tests/e2e/                  │    │       │
│   │  │  │          + sprint-status 업데이트          │    │       │
│   │  │  │                                          │    │       │
│   │  │  │ 실패:                                     │    │       │
│   │  │  │ → 원인 분석 + failure tag 분류             │    │       │
│   │  │  │ → regression story 문서 생성               │    │       │
│   │  │  │ → "failed" + regression-stories/          │    │       │
│   │  │  │           + sprint-status 업데이트          │    │       │
│   │  │  │                                          │    │       │
│   │  │  │ → 종료 (컨텍스트 소멸)                     │    │       │
│   │  │  └──────────────┬───────────────────────────┘    │       │
│   │  │                 │                                │       │
│   │  │  orchestrate 분기:                                │       │
│   │  │  ├─ "passed" → 2c로 진행                         │       │
│   │  │  └─ "failed" → 2a로 돌아감 (regression story 포함)│       │
│   │  │                 │                                │       │
│   │  │                 ▼                                │       │
│   │  │  2c. 에픽 통합 리뷰 + 사람 개입 ②                 │       │
│   │  │  ┌──────────────────────────────────────────┐    │       │
│   │  │  │ bf-lead-review  (스폰)                    │    │       │
│   │  │  │ 조율 패턴: discourse                      │    │       │
│   │  │  │ 초기 로딩: tech-spec, conventions, 전체 diff│    │       │
│   │  │  │ 모델: 에픽 내 L/XL 있으면 Opus, 없으면 Sonnet│   │       │
│   │  │  │                                          │    │       │
│   │  │  │  에픽 전체를 대상으로 리뷰                  │    │       │
│   │  │  │  - 스토리 간 중복/불일치                    │    │       │
│   │  │  │  - 아키텍처 드리프트                       │    │       │
│   │  │  │  - convention 위반                        │    │       │
│   │  │  │                                          │    │       │
│   │  │  │  Rev-A ◄──► Rev-B ◄──► Rev-C             │    │       │
│   │  │  │    └── 직접 대화로 합의 시도 ─┘            │    │       │
│   │  │  │           │                              │    │       │
│   │  │  │      미합의 → Lead 중재                   │    │       │
│   │  │  │           │                              │    │       │
│   │  │  │      그래도 미합의 → 버린다 (기록)         │    │       │
│   │  │  │                                          │    │       │
│   │  │  │  review.md 생성 (합의/미합의 분리)         │    │       │
│   │  │  │                                          │    │       │
│   │  │  │  ── 사람 개입 ② ──────────────────────── │    │       │
│   │  │  │  리뷰어 에이전트와 함께 최종 리뷰           │    │       │
│   │  │  │  미합의 쟁점 일괄 결정                     │    │       │
│   │  │  │                                          │    │       │
│   │  │  │  승인 시:                                 │    │       │
│   │  │  │  → "승인" → orchestrate에 전달            │    │       │
│   │  │  │                                          │    │       │
│   │  │  │  수정 지시 시:                             │    │       │
│   │  │  │  → review.md에 수정 지시 원문 기록         │    │       │
│   │  │  │    (Story별 구체적 피드백 + 근거)           │    │       │
│   │  │  │  → "수정 지시" + review.md → orchestrate   │    │       │
│   │  │  │                                          │    │       │
│   │  │  │ → 종료 (컨텍스트 소멸)                     │    │       │
│   │  │  └──────────────┬───────────────────────────┘    │       │
│   │  │                 │                                │       │
│   │  │  orchestrate 분기:                                │       │
│   │  │  ├─ "승인" → 에픽 완료                            │       │
│   │  │  └─ "수정 지시"                                   │       │
│   │  │      해당 Story → in_progress로 변경               │       │
│   │  │      나머지 Story → done 유지                      │       │
│   │  │      │                                            │       │
│   │  └──────┘  ← 2a로 돌아감 (같은 경로)                  │       │
│   │                                                     │       │
│   │  승인 → 에픽 완료                                    │       │
│   └─────────────────────────────────────────────────────┘       │
│                     │                                           │
│                     ▼ 다음 에픽 있으면 Epic 루프 반복             │
│                     │ 없으면 ↓                                   │
│                                                                 │
│ → "done" + sprint-status.yaml (전체 에픽 완료)                   │
│ → 종료 (컨텍스트 소멸)                                           │
└─────────────────────────────────────────────────────────────────┘
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

## Orchestrate 분기 요약

orchestrate는 분석하지 않는다. 수신한 상태에 따라 분기만 한다.

| 수신 | 다음 행동 |
|------|----------|
| lead-plan: "done" | 사용자 확인 → 에픽 루프 시작 |
| lead-implement: "done" | E2E agent 트리거 |
| lead-implement: "done (stuck)" | 사람에게 판단 요청 (stuck.md 참조) |
| E2E agent: "passed" | lead-review 트리거 |
| E2E agent: "failed" | 2a로 돌아감 (regression story 포함) |
| lead-review: "승인" | 에픽 완료, 다음 에픽 |
| lead-review: "수정 지시" | 2a로 돌아감 (review.md 참조) |

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
```

---

## 에스컬레이션 프로토콜

"stuck"은 중간 과정이 아니라 최종 상태이다. 각 층은 자기 범위에서 판단하고, 판단이 필요하면 파일과 함께 위로 올린다.

```
story agent
  → "stuck" + stuck.md (시도한 접근들, 에러 내용 기록)
  → 종료

bf-lead-implement
  → 다른 Story들은 계속 진행
  → 모든 Story 완료 후 orchestrate에 보고:
    "done (stuck: Story-N)" + sprint-status.yaml + stuck.md
  → 종료

orchestrate
  → stuck Story가 있으면 E2E/리뷰로 진행하지 않음
  → 사람에게 판단 요청 (stuck.md 참조)

사람
  ├─ AC 수정 → 해당 Story 재시도 (2a로)
  ├─ 접근 변경 지시 → story 문서 수정 후 재시도
  ├─ Story 스킵 → 에픽에서 제외하고 진행
  └─ 에픽 재설계 → lead-plan으로 돌아감
```

---

## 수정 지시 전달 프로토콜

수정 지시는 대화로 전달하지 않는다. 파일(review.md)이 원본을 운반한다.

```
사람: "Story-2 에러 핸들링 바꿔"
  → bf-lead-review가 review.md에 수정 지시 원문 기록 → 종료
    → orchestrate: "Story-2 수정, 참조: review.md" (경로만 전달)
      → bf-lead-implement: review.md 직접 읽음
        → story agent: review.md 직접 읽음
```

review.md 수정 지시 섹션 형식:

```markdown
## 수정 지시 (사람 개입 ② 결과)

### Story-2
- 에러 핸들링: try-catch 대신 Result 패턴으로 변경
- 근거: 리뷰어 B의 지적 + 사람 판단

### Story-3
- 네이밍: getUserData → fetchUserProfile로 통일
- 근거: conventions.md 위반
```

모든 단계가 같은 파일을 읽는다. 요약이 끼어들 여지가 없다.

---

## sprint-status.yaml 쓰기 권한

각 단계가 순차적이므로 동시 쓰기가 발생하지 않는다.
story agent는 sprint-status.yaml을 만지지 않는다.

| 에이전트 | sprint-status.yaml 권한 | 쓰기 내용 |
|----------|------------------------|----------|
| story agent | 없음 (코드+커밋만) | — |
| bf-lead-implement | 쓰기 | story 상태, 메트릭 (retries, approaches, stuck) |
| E2E agent | 쓰기 | e2e 상태, failure tag, regression story 추가 |
| bf-lead-review | 쓰기 | review 상태, blocker/recommended 수 |
| orchestrate | 읽기만 | — |

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

---

## 컨텍스트 수명

```
  Main Session ·······························→ (세션 끝까지 유지)
       │                                        컨텍스트: 최소
       │
       ├─ Orchestrate ────────────────┤ 종료    컨텍스트: 중간
       │       │                                (에픽 루프 동안만)
       │       │                                "done" 수신 + 분기만
       │       │
       │       ├─ Lead-Plan ────┤ 종료
       │       │     ├─ Creator-A ┤ 종료
       │       │     └─ Creator-B ┤ 종료
       │       │
       │       ├─ Lead-Impl ────┤ 종료          (에픽당 1회+)
       │       │     ├─ Agent-S ───┤ 종료        Lead는 "done" 수신만
       │       │     ├─ Agent-M ───┤ 종료        코드를 직접 만지지 않음
       │       │     ├─ Impl-A ────┤ 종료
       │       │     └─ Impl-B ────┤ 종료
       │       │
       │       ├─ E2E Agent ────┤ 종료          (에픽당 1회+)
       │       │
       │       └─ Lead-Review ──┤ 종료          (에픽당 1회+)
       │             │                          discourse + 사람 개입 ②
       │             ├─ Rev-A ────┤ 종료         까지 포함 후 종료
       │             └─ Rev-B ────┤ 종료
       │
       ├─ archive (직접) ─┤
       ├─ metrics (직접) ─┤
       └─ conventions (직접) ─┤

  ────────────────────────────────────────→ 시간
  │← 파일만 남는다 →│
```

---

## 사람 개입 요약

| 개입 | 시점 | 내용 |
|------|------|------|
| ① | Tech Spec 리뷰 후 | 승인 / 수정요청. 에픽/스토리 생성 직후 난이도 태그 확인 |
| ② | 에픽 통합 리뷰 | bf-lead-review 내에서 리뷰어 에이전트와 협업. 승인 → 에픽 완료. 수정 지시 → review.md에 기록 → 해당 Story 재구현 → 같은 흐름 반복 |
| stuck | 구현 중 (비정기) | bf-lead-implement가 stuck 보고 시 orchestrate 경유로 사람에게 판단 요청 |

전체 워크플로우 사람 개입 횟수: **① + (에픽 수 × ②) + (stuck 발생 시)**

---

## Lead 스킬 요약

| Lead | 조율 패턴 | 초기 로딩 | 역할 | 모델 |
|------|-----------|----------|------|------|
| bf-lead-orchestrate | sequence | tech-spec, sprint-status | "done" 수신 → 분기 → 다음 Lead 트리거 | 항상 Opus |
| bf-lead-plan | distribute | tech-spec, conventions | 에픽/스토리 구조 결정, 병렬 분배 + 취합 | 항상 Opus |
| bf-lead-implement | monitor | story 문서, conventions, review.md(수정 시) | agent 스폰 + "done"/"stuck" 수신 + sprint-status 업데이트 | Opus/Sonnet |
| bf-lead-review | discourse | tech-spec, conventions, 전체 diff | 리뷰어 토론 + 사람 개입 ② + 결정 전달 | Opus/Sonnet |
