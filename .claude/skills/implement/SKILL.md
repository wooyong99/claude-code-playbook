---
name: implement
description: 요구사항을 마일스톤 단위로 분해하여 설계 문서 작성(D) → 코드 작성(A) → 아키텍처 검토(B) → 수정 루프를 오케스트레이션하는 스킬. 기능 구현, 리팩토링, UseCase 추가, 도메인 모델 변경 등 코드 작업이 필요한 모든 상황에서 사용한다. "구현해줘", "만들어줘", "추가해줘", "리팩토링", "implement", "/implement" 같은 요청에 이 스킬을 사용한다.
---

# implement — 코드 작성 및 검토 오케스트레이션

사용자의 구현/리팩토링 요구사항을 **마일스톤 단위로 분해**하고, 각 마일스톤에 대해 **기술설계 서브에이전트(D)**, **코드 작성 서브에이전트(A)**, **아키텍처 검토 서브에이전트(B)** 를 오케스트레이션한다.

---

## 당신(메인 Claude)의 역할 — 오케스트레이터

| 주체 | 책임 |
|------|------|
| **당신 (메인 Claude)** | 요구사항 분석, 마일스톤 분할, D·A·B 위임, 반복 종료 판단, 사용자 보고 |
| **Agent D** (`subagent_type="design-writer"`) | 마일스톤별 기술설계문서(TDD) 작성. `write-tech-design-doc` 스킬 사용 |
| **Agent A** (`subagent_type="code-writer"`) | Kotlin 코드 작성·수정, 테스트, 빌드 확인 |
| **Agent B** (`subagent_type="architecture-reviewer"`) | `docs/backend/architecture/*`·`docs/backend/policies/*` 준수 검토만. Read/Glob/Grep만 보유, 수정 권한 없음 |

**에이전트 계약 문서** (입출력 규격의 단일 출처):
- D: [`references/design-writer-contract.md`](references/design-writer-contract.md)
- A: [`references/code-writer-contract.md`](references/code-writer-contract.md)
- B: [`references/architecture-reviewer-contract.md`](references/architecture-reviewer-contract.md)

### 절대 지켜야 할 제약

1. **당신은 파일을 직접 Read/Edit/Write 하지 않는다.** 코드 작업은 전부 A에게, 설계 문서는 D에게 위임.
   - 예외: 사용자 요구사항 이해에 필요한 경우 `docs/backend/README.md` 같은 **맵 문서 하나 정도** Read는 허용.
2. **당신은 검토를 직접 수행하지 않는다.** 검토는 전부 B에게 위임.
3. **D·A·B는 서로 호출하지 않는다.** 모든 통신은 당신을 경유.
4. **B 호출은 매번 새 인스턴스**로 수행 (fresh context).
5. **D 호출은 매번 새 인스턴스**로 수행 (fresh context). **A 호출은 마일스톤 첫 호출만 새 인스턴스**로 수행. 같은 마일스톤 내 재호출(위반 수정, 체크포인트 재개)은 동일 인스턴스를 이어서 사용한다.

---

## Phase 0: 요구사항 분석

1. 사용자 입력을 한 문장으로 재진술하여 이해 확인.
2. 애매하거나 중요한 정보가 누락되면 **사용자에게 확인 질문 후 대기**. 임의 추정하지 않음.
3. 이해가 선명하면 Phase 1로.

## Phase 1: 마일스톤 분할

분할 기준:
- 독립적으로 검토 가능한가 (B가 한 번에 의미 있는 검토 가능)
- 독립적으로 커밋 가능한가
- A의 컨텍스트가 감당 가능한가 (파일 10~15개 범위 권장)

규모별 기준:
- 단일 CRUD / 단일 UseCase 추가 → **1 마일스톤**
- 여러 도메인 걸친 기능 → **2~4 마일스톤**
- 대규모 리팩토링 / 신규 서브시스템 → **5+ 마일스톤** (사용자에게 사전 확인)

각 마일스톤을 `TaskCreate`로 등록하고, 사용자에게 분할 결과 출력:

```
📋 마일스톤 계획
  M1/{total}: <제목>
  M2/{total}: <제목>
  ...
```

규모가 크면 (5개 이상 마일스톤) 사용자에게 진행 여부 확인.

진행이 확정되면 아래 명령으로 `run_id`를 생성·보관한다:
```bash
mkdir -p .claude/checkpoints && run_id=$(date +%Y%m%d-%H%M)
```
이 값은 Phase 2 전체에서 에이전트 프롬프트의 `[체크포인트 파일]` 경로를 구성하는 데 사용된다.

---

## Phase 2: 마일스톤별 순차 실행

각 마일스톤에 대해 아래 루프를 수행한다. `iter`는 해당 마일스톤의 A↔B 반복 횟수.

### Step 2-1. 시작 알림

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 [M{n}/{total}] {마일스톤 제목} — 시작
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

해당 task를 `TaskUpdate`로 `in_progress`로 전환.

### Step 2-2. Agent D 위임 (기술설계 문서 작성)

```
📝 기술설계 문서 작성 중... (Agent D)
```

`Agent` 도구 호출:
- `subagent_type`: `design-writer`
- `description`: `[M{n}] 기술설계 문서 작성`
- `prompt`: [`references/design-writer-contract.md`](references/design-writer-contract.md) — Input 섹션 형식으로 구성. 현재 마일스톤의 제목·요구사항·관련 레이어·도메인을 채운다. `[체크포인트 파일]` 필드에 `.claude/checkpoints/design-writer-{run_id}-M{n}.md`를 포함한다.

D의 응답을 받으면:

**응답 첫 줄이 `CONTEXT_CHECKPOINT:` 신호인 경우:**
```
⚡ [M{n}] 컨텍스트 체크포인트 저장됨 — D 재호출 중...
```
체크포인트 파일을 Read한 뒤, D를 재호출한다. 재호출 프롬프트에 아래 필드를 추가:
```
[체크포인트]: .claude/checkpoints/design-writer-{run_id}-M{n}.md 참조. 이어서 작업 진행.
```

**응답 첫 줄이 `TDD_CREATED:` 인 경우:**
```
✅ TDD 작성 완료 · {tdd-path}
```
TDD 파일 경로와 설계 요약을 변수로 보관 (A에게 전달해야 함).

**응답 첫 줄이 `TDD_SKIPPED:` 인 경우:**
```
⏭️  TDD 스킵 · {reason}
```

→ Step 2-3으로.

### Step 2-3. Agent A 위임 (코드 작성)

```
🔨 코드 작성 위임 중... (Agent A)
```

`Agent` 도구 호출:
- `subagent_type`: `code-writer`
- `description`: `[M{n}] 코드 작성`
- `prompt`: [`references/code-writer-contract.md`](references/code-writer-contract.md) — Input > Case A 형식으로 구성. 현재 마일스톤의 제목·요구사항·관련 레이어·도메인을 채우고, D 결과에 따라 `[기술설계문서]` 필드를 포함하거나 생략한다. `[체크포인트 파일]` 필드에 `.claude/checkpoints/code-writer-{run_id}-M{n}.md`를 포함한다.

A의 응답을 받으면:

**응답 첫 줄이 `CONTEXT_CHECKPOINT:` 신호인 경우:**
```
⚡ [M{n}] 컨텍스트 체크포인트 저장됨 — 재호출 중...
```
체크포인트 파일을 Read한 뒤, 아래 형식으로 A를 재호출한다 (Case A 템플릿에 추가):
```
[체크포인트]: .claude/checkpoints/code-writer-{run_id}-M{n}.md 참조. 완료된 작업은 건너뛰고 남은 작업부터 이어서 수행.
```
재호출 후 응답을 받을 때까지 이 처리를 반복한다.

**정상 응답인 경우:**
```
✅ 코드 작성 완료 · 변경 파일 {N}개
```

변경 파일 목록을 변수로 보관 (B에게 전달해야 함).

### Step 2-4. Agent B 위임 (아키텍처 검토)

```
🔍 아키텍처 규칙 검토 중... (Agent B, 시도 {iter}/5)
```

`Agent` 도구 호출:
- `subagent_type`: `architecture-reviewer`
- `description`: `[M{n}] 규칙 검토`
- `prompt`: [`references/architecture-reviewer-contract.md`](references/architecture-reviewer-contract.md) — Input 섹션 형식으로 구성. A가 반환한 변경 파일 목록과 핵심 설계 결정을 채운다. `[체크포인트 파일]` 필드에 `.claude/checkpoints/arch-reviewer-{run_id}-M{n}-r{iter}.md`를 포함한다.

B의 응답을 받으면 아래 포맷으로 즉시 콘솔에 출력:

```
📋 검토 결과 (시도 {iter}/5):
  상태: ✅ PASS  또는  ⚠️ 위반 {K}건
  {위반이 있을 경우 각 항목을 한 줄씩 — 파일명 + 위반 규칙 요약}
```

B의 응답 처리:
- **응답 첫 줄이 `CONTEXT_CHECKPOINT:` 신호인 경우**: 체크포인트 파일을 Read한 뒤 B를 재호출. 재호출 프롬프트에 `[체크포인트]: .claude/checkpoints/arch-reviewer-{run_id}-M{n}-r{iter}.md 참조. 완료된 파일은 건너뛰고 남은 파일부터 이어서 검토.` 를 추가. 완료된 파일의 위반 결과는 이전 응답에서 누적해서 보관.
- 응답이 `PASS`면 → Step 2-7으로 (마일스톤 완료)
- 위반 YAML이면 → Step 2-5로

### Step 2-5. 위반 수정 위임 (A 재호출)

```
⚠️  위반 {K}건 발견 — 수정 적용 중... (Agent A, 시도 {iter}/5)
```

`iter`를 1 증가. **iter > 5이면 Step 2-8 (escalation)로**.

`Agent` 도구 호출:
- `subagent_type`: `code-writer`
- `description`: `[M{n}] 위반 수정`
- `prompt`: [`references/code-writer-contract.md`](references/code-writer-contract.md) — Input > Case B 형식으로 구성. B가 반환한 위반 YAML을 채운다. `[체크포인트 파일]` 필드에 `.claude/checkpoints/code-writer-{run_id}-M{n}.md`를 포함한다 (마일스톤 내 A 인스턴스를 이어 사용하므로 Case A와 동일 경로).

A의 응답을 받으면:
```
✅ 수정 완료 · 적용 {S}건 · 실패 {F}건
```

실패가 있으면 상세를 사용자에게 노출. → Step 2-6 (재검토).

### Step 2-6. 재검토 (Step 2-4 반복)

```
🔍 재검토 중... (Agent B, 시도 {iter}/5)
```

Step 2-4와 동일한 B 위임. B의 응답을 받으면 Step 2-4와 동일한 포맷으로 콘솔에 출력한 뒤 처리:
- `PASS` → Step 2-7
- 위반 여전 → Step 2-5로 돌아가서 반복

### Step 2-7. 마일스톤 완료

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 [M{n}/{total}] {제목} 완료 ({iter} 라운드)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

해당 task를 `TaskUpdate`로 `completed`로 전환. 다음 마일스톤으로.

### Step 2-8. Escalation (5회 초과 시)

```
⛔ [M{n}] 자동 수렴 실패 — A↔B 루프 5회 초과

마지막 검토 결과 요약:
{B의 마지막 응답 요약 3~5줄}

선택지:
  1. 현재 상태로 커밋하고 수동 검토
  2. 메인 Claude가 직접 개입하여 수정
  3. 요구사항을 재정의하고 해당 마일스톤 재시작
  4. 해당 마일스톤을 스킵하고 다음으로 진행
  5. 전체 중단
```

**사용자 선택 대기**. 사용자 응답에 따라 처리:
- 1, 2, 3, 4, 5 각각에 맞는 후속 흐름 실행.
- 2번 선택 시에만 메인 Claude가 직접 Edit (이 경우에 한해 예외적으로 허용).

---

## Phase 3: 전체 완료 보고

모든 마일스톤이 완료되면 최종 요약:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✨ 구현 완료

마일스톤: {완료}/{총}
총 A↔B 라운드: {합계}
마일스톤별 라운드: M1={r1}, M2={r2}, ...

변경 파일 총 {N}개. 다음 단계 제안:
  1. `smart-commit` 스킬로 마일스톤 단위 커밋 그룹핑
  2. 또는 바로 커밋 작성
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 요약 — 한 번에 보기

```
[Phase 0] 요구 분석 → 불명확하면 사용자 확인
[Phase 1] 마일스톤 분할 → TaskCreate
[Phase 2] 각 M마다:
    D 위임(설계) →
      TDD_CREATED → A 위임(구현, TDD 참조) → B 위임(검토)
      TDD_SKIPPED → A 위임(구현)            → B 위임(검토)
        PASS  → 다음 M
        위반  → A 위임(수정) → B 재검토 → ... (max 5 iter)
        5회↑ → 사용자 escalate
[Phase 3] 완료 보고 → 커밋 제안
```
