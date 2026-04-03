---
name: start-workflow
description: "전체 개발 워크플로우를 자동화. 요청 분석 → 난이도 산정 → Plan 리뷰 → 구현 → 품질 루프 → 문서 동기화 → PR → 성찰까지 일관된 파이프라인으로 실행한다."
allowed-tools: AskUserQuestion, Read, Write, Edit, Glob, Grep, Bash, Agent, EnterPlanMode, ExitPlanMode, Skill
argument-hint: <작업 설명 또는 빈 값>
user-invocable: true
---

# Start Workflow — Orchestrator

전체 개발 라이프사이클을 **오케스트레이션 패턴**으로 실행한다.
각 자율 실행 Phase를 전용 서브 에이전트에 위임하여, 단일 컨텍스트 소진 없이 전 단계를 완주한다.

```
[유저 대화] Phase 0~3  : 직접 실행 (Spec, Plan, 리뷰)
[상태 저장] Phase 3.5  : 상태 파일 생성
[자율 실행] Phase 4~8  : 서브 에이전트 순차 위임
[유저 대화] Phase 9    : 최종 보고 + 보완점 적용
```

## Language Rule

유저와의 모든 대화는 **한국어**로 진행한다.

## 자율 실행 규칙

- Phase 0~3: 유저와 대화하며 Spec 확인, Plan 리뷰 피드백 반영
- **Phase 3.5 이후**: 서브 에이전트를 순차 호출하며 **묻지 않고 자동 실행**
- Phase 9: 최종 보고에서 보완점 반영 여부만 유저에게 질문

---

## Phase 0: 작업 범위 수집

`/minmo-s-harness:request`를 호출하여 Technical Spec을 생성한다.

- `$ARGUMENTS`가 있으면 request 스킬에 전달한다.
- request 스킬이 완료되면 생성된 **Technical Spec** 전문을 보관한다.
- Spec에서 **작업 유형** (생성/수정/검토/디버깅)을 확인한다.

> Spec을 유저에게 보여주고 확인을 받는다.

---

## Phase 1: 난이도 산정

Technical Spec을 분석하여 1~10 난이도를 산정한다.

| 요소 | 낮음 (1-3) | 중간 (4-6) | 높음 (7-10) |
|------|-----------|-----------|------------|
| 파일 수 | 1-3개 | 4-7개 | 8개+ |
| 레이어 | 단일 | 2개 | 3개 전체 |
| DB 변경 | 없음 | 컬럼 추가 | 신규 테이블 |
| 외부 연동 | 없음 | 기존 gRPC | 신규 gRPC |
| 비즈니스 복잡도 | 단순 CRUD | 조건 분기 3개 이하 | 상태 머신 |
| 엣지 케이스 | 1-2개 | 3-5개 | 6개+ |

출력: `난이도: [N]/10 — [근거]`

> 난이도 7+: Phase 3에서 Codex 리뷰 추가.

---

## Phase 2: Scope Reviewer 준비

Phase 5에서 사용할 scope-reviewer 정보를 메모한다:
- Technical Spec 전문
- 엣지 케이스 목록

---

## Phase 3: Plan 작성 + 리뷰

### 3.1 Plan 작성

`EnterPlanMode` 활성화. Plan 포함 내용:
- 구현 순서 (파일 단위)
- 각 파일 변경 내용 요약
- 의존 관계
- 예상 리스크

### 3.2 다관점 Plan 리뷰

**최대 3개 서브에이전트 병렬 실행**, 2배치로 진행:

```
Batch 1 (병렬): 유지보수성 + 성능 + 엣지 케이스
Batch 2 (병렬): 데이터 정합성 + 보안 + 기존 코드 영향
```

각 에이전트(subagent_type: `general-purpose`) 프롬프트:

```
당신은 [관점명] 리뷰어입니다.
아래 Plan을 [관점] 관점에서만 리뷰하세요.

## Technical Spec
[Spec 전문]

## Plan
[Plan 전문]

## 리뷰 형식
**Verdict**: APPROVE / CONCERN / REJECT
**Issues**: [문제 목록 또는 "없음"]
**Suggestions**: [개선 제안 또는 "없음"]
```

리뷰 종합:
- **REJECT 1개+**: Plan 수정 → 해당 관점 재리뷰
- **CONCERN만**: 타당한 것 자동 반영

### 3.3 Codex 리뷰 (난이도 7+)

Codex 사용 가능 시 Architect 관점으로 Plan 리뷰를 위임한다.
불가 시 건너뛴다.

### 3.4 Plan 확정

`ExitPlanMode` 실행.

---

## Phase 3.5: 상태 파일 생성 + 자율 실행 시작

Write tool로 `/tmp/workflow-state.md`를 생성한다:

```markdown
# Workflow State

## Spec
[Technical Spec 전문 그대로 복사]

## Task Type
[생성/수정/검토/디버깅]

## Difficulty
[N]/10

## Edge Cases
[엣지 케이스 목록]

## Plan
[확정된 Plan 전문 그대로 복사]
```

출력: **"자율 실행을 시작합니다. Phase 4~8을 서브 에이전트로 순차 실행합니다."**

---

## Phase 4~8: 서브 에이전트 순차 실행

**각 Phase를 전용 서브 에이전트에 위임한다.**
이전 에이전트가 완료된 후 다음 에이전트를 실행한다.
각 에이전트의 반환 결과를 기록해 둔다 (Phase 9 보고서에 사용).

### Phase 4: 구현

```
Agent tool:
  subagent_type: minmo-s-harness:workflow-implementer
  prompt: |
    상태 파일 `/tmp/workflow-state.md`를 읽고 Plan에 따라 코드를 구현하세요.
    프로젝트 루트: {현재 작업 디렉토리}
    구현 완료 후 변경 파일 목록, 커밋 수, Plan 대비 차이점을 보고하세요.
```

완료 후 유저에게 간략 보고: "Phase 4 완료: [변경 파일 수]개 파일, [커밋 수]개 커밋"

### Phase 5: 품질 루프

```
Agent tool:
  subagent_type: minmo-s-harness:workflow-quality-loop
  prompt: |
    상태 파일 `/tmp/workflow-state.md`를 읽고 품질 루프를 실행하세요.
    프로젝트 루트: {현재 작업 디렉토리}
    루프 결과(반복 횟수, 각 단계 수정 건수, 최종 판정)를 보고하세요.
```

완료 후: "Phase 5 완료: [루프 횟수]회, 총 [수정 건수]건 수정"

### Phase 6: 문서 동기화 (조건부)

**작업 유형이 API 생성/수정/삭제인 경우만 실행. 그 외는 건너뛴다.**

```
Agent tool:
  subagent_type: minmo-s-harness:workflow-doc-sync
  prompt: |
    상태 파일 `/tmp/workflow-state.md`를 읽고 API 문서를 동기화하세요.
    작업 유형: {Task Type}
    프로젝트 루트: {현재 작업 디렉토리}
```

### Phase 7: PR

```
Agent tool:
  subagent_type: minmo-s-harness:workflow-pr
  prompt: |
    상태 파일 `/tmp/workflow-state.md`를 읽고 PR을 생성하세요.
    프로젝트 루트: {현재 작업 디렉토리}
    PR URL을 반드시 보고하세요.
```

### Phase 8: 성찰

```
Agent tool:
  subagent_type: minmo-s-harness:workflow-reflection
  prompt: |
    상태 파일 `/tmp/workflow-state.md`를 읽고 워크플로우 성찰을 수행하세요.
    프로젝트 루트: {현재 작업 디렉토리}
    성찰 결과와 스킬 보완점을 보고하세요.
```

---

## Phase 9: 최종 보고

Phase 4~8 에이전트들의 결과를 종합하여 보고서를 작성한다.

```markdown
## Workflow Report

### 1. 작업 요약
- **작업 유형**: [생성/수정/검토/디버깅]
- **난이도**: [N]/10 (산정) → [M]/10 (체감)
- **PR**: [PR URL]

### 2. 구현 내역
- **변경 파일**: [N]개
- **커밋 수**: [N]개
- **핵심 로직**: [요약]

### 3. 엣지 케이스 대응
| # | 케이스 | 대응 방법 |
|---|--------|----------|

### 4. 품질 루프 결과
| 단계 | 루프 횟수 | 수정 건수 |
|------|----------|----------|
| simplify | N | M |
| convention | N | M |
| e2e | N | M |
| scope-review | N | M |

### 5. 문서 동기화
- Apidog 업데이트: [Y/N, 요약]

### 6. 성찰
[성찰 에이전트 결과]

### 7. 보완점
| # | 대상 스킬 | 보완 내용 | 적용 여부 |
|---|----------|----------|----------|
```

### 보완점 적용

> "위 보완점을 해당 스킬에 반영할까요? (전체/선택/건너뛰기)"

- **전체**: 모든 보완점을 해당 스킬 파일에 반영한다.
- **선택**: 유저가 번호로 선택한 항목만 반영한다.
- **건너뛰기**: 보고서만 출력하고 종료한다.

### 정리

상태 파일을 삭제한다:

```bash
rm -f /tmp/workflow-state.md
```

---

## 흐름 요약

```
[유저 대화]
Phase 0: /request → Technical Spec (유저 확인)
Phase 1: 난이도 산정 (1-10)
Phase 2: scope-reviewer 메모
Phase 3: Plan 작성 → 6관점 리뷰 (3+3 병렬) → [난이도 7+: Codex] → Plan 확정
Phase 3.5: 상태 파일 생성 → "자율 실행 시작"

[서브 에이전트 순차 실행 — 유저 확인 없이 완주]
Phase 4: workflow-implementer    → 구현 + 커밋
Phase 5: workflow-quality-loop   → 품질 루프 (최대 3회)
Phase 6: workflow-doc-sync       → 문서 동기화 (API 변경 시만)
Phase 7: workflow-pr             → PR 생성
Phase 8: workflow-reflection     → 성찰

[유저 대화]
Phase 9: 최종 보고 → 보완점 적용 (유저 선택) → 정리
```
