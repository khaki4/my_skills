# 참고 기술 및 방법론

BF 워크플로우는 다음 외부 기술·방법론을 참고·차용하여 설계되었다. 각 기술의 영향 범위를 명시하여 향후 업데이트 판단에 활용한다.

## 외부 참조

| 기술 | 요약 | BF 적용 범위 | 출처 |
|------|------|-------------|------|
| **BMAD Method** | 멀티 에이전트 AI 개발 프레임워크. 12+ 페르소나가 기획→구현까지 순차 누적 | Phase 2의 Epic/Story 분해 구조, 페르소나 기반 에이전트 설계 | [bmad-code-org/BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD) |
| **OCR (Open Code Review)** | 다관점 AI 코드 리뷰. 독립 분석 → discourse → 합의/미합의 분리 | bf-lead-review의 리뷰 패턴 전체. Convention Guard + 추가 리뷰어 구조 | [spencermarx/open-code-review](https://github.com/spencermarx/open-code-review) |
| **Gastown** | Steve Yegge의 멀티 에이전트 워크스페이스 매니저. git-backed 영속성, 역할 기반 조율 | 에이전트 조율 아키텍처 참고 (BF는 파일 기반 컨텍스트 소멸 방식으로 차별화) | [steveyegge/gastown](https://github.com/steveyegge/gastown) |
| **Ralph Loop** | Geoffrey Huntley의 반복적 AI 에이전트 패턴. "완료될 때까지 루프" | bf-lead-implement의 TDD 사이클 기반. BF는 가드레일 추가 (max 5회, stuck detection, 접근 전환) | [ghuntley.com/loop](https://ghuntley.com/loop/) |
| **Superpowers** | Jesse Vincent의 코딩 에이전트 스킬 프레임워크. 자동 스킬 탐색 + 실행 | `<HARD-GATE>` 패턴 차용 (v4.3.0). 에이전트 단계 건너뛰기 방지 기법 | [obra/superpowers](https://github.com/obra/superpowers) |
| **TDD (Red-Green-Refactor)** | Kent Beck의 테스트 주도 개발. 테스트 먼저 → 실패 확인 → 구현 → 통과 | Ralph Loop의 핵심 사이클. 모든 Story에 적용 | 표준 소프트웨어 공학 |

## BF 오리지널

| 개념 | 요약 |
|------|------|
| **Convention Guard** | conventions.md 기반 필수 리뷰어 역할. append-only 규칙 축적으로 스프린트마다 자기 강화 |
| **Discourse 에스컬레이션** | OCR discourse를 확장한 3단계 프로토콜: 직접 대화 → Lead 중재 → 버림(기록). 무한 토큰 소비 방지 |
| **컨텍스트 소멸 계약** | 각 층이 "done" + 파일로 종료, 컨텍스트는 소멸. Gastown의 영속 방식과 반대 접근 |
| **단일 쓰기 지점** | sprint-status.yaml 동시 쓰기 방지. 단계별 담당 Lead만 기록 |
| **사람 = 느린 에이전트** | 사람 판단 지점을 외부 경계(bf-execute)로 제한. 정확히 2개: Spec 승인 + Epic 결과 확인 |
