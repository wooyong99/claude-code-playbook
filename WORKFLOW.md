# WORKFLOW — `implement` 스킬 운용 흐름

`implement` 스킬은 이 리포지토리의 **주력 코드 작업 스킬**이다.
요구사항 한 줄을 받아서, 마일스톤 단위 분할 → 설계 → 구현 → 아키텍처 검토 → 자동 수정 루프를 돌려 PASS까지 수렴시킨다.

---

## 한눈에 보기

```
[사용자] "<요구사항>"  (implement 스킬이 자동 트리거)
   ↓
[메인 Claude] Phase 0 요구 분석 → 불명확하면 사용자 확인
   ↓
[메인 Claude] Phase 1 마일스톤 분할 → TaskCreate
   ↓
각 마일스톤 M마다:
   ┌──────────────────────────────────────────────┐
   │ [메인] Agent D 위임 (TDD 작성)                │
   │    ↓ TDD_CREATED / TDD_SKIPPED               │
   │ [메인] Agent A 위임 (구현)                    │
   │    ↓                                         │
   │ [메인] Agent B 위임 (검토)                    │
   │    ↓                                         │
   │ PASS?  ── 예 ──→ 다음 마일스톤                │
   │    │                                         │
   │    아니오                                     │
   │    ↓                                         │
   │ [메인] Agent A 재위임 (위반 수정)             │
   │    ↓                                         │
   │ [메인] Agent B 재검토                         │
   │    ↓                                         │
   │ (최대 5회 반복 — 초과 시 사용자 escalate)      │
   └──────────────────────────────────────────────┘
   ↓
[메인 Claude] Phase 3 완료 보고 → 커밋 제안
```

---

## 등장 인물

| 역할 | 주체 | 책임 |
|------|------|------|
| **오케스트레이터** | 메인 Claude | 요구 분석, 마일스톤 분할, D·A·B 위임, 종료 판단, 사용자 보고 |
| **Agent D** | `design-writer` 서브에이전트 | 마일스톤별 기술설계문서(TDD) 작성. 코드 수정 금지. |
| **Agent A** | `code-writer` 서브에이전트 | Kotlin 코드 작성·수정·테스트·빌드 |
| **Agent B** | `architecture-reviewer` 서브에이전트 | 문서 기반 규칙 준수 검토. Read 전용. |

### 절대 지켜야 할 제약 (스킬에 명시)

1. **메인 Claude는 파일을 직접 편집하지 않는다.** 코드 작업은 A에게, 설계 문서는 D에게 위임.
2. **메인 Claude는 검토를 직접 하지 않는다.** 검토는 전부 B에게 위임.
3. **D·A·B는 서로 호출하지 않는다.** 모든 통신은 메인을 경유.
4. **B 호출은 매번 새 인스턴스**로 수행 (fresh context).
5. **D 호출도 매번 새 인스턴스**로 수행. **A는 마일스톤 첫 호출만 새 인스턴스**, 같은 마일스톤 내 수정 재호출은 동일 인스턴스를 이어 사용한다.

---

## Phase 0 — 요구사항 분석

메인이 사용자 입력을 받으면:

1. **한 문장으로 재진술**해 이해를 확인
2. 애매하거나 정보가 부족하면 **사용자에게 확인 질문 후 대기** (임의 추정 금지)
3. 이해가 선명하면 Phase 1로

> 사용자 입장에서 바라는 것: "먼저 묻고 나서 움직인다"는 신뢰. 추정이 많을수록 재작업이 늘어난다.

---

## Phase 1 — 마일스톤 분할

분할 기준:

- **독립적으로 검토 가능한가** — B가 한 번에 의미 있는 검토 가능
- **독립적으로 커밋 가능한가** — 되돌리기·분리 이식 가능
- **A의 컨텍스트가 감당 가능한가** — 파일 10~15개 이내 권장

규모별 기준:

| 규모 | 마일스톤 수 |
|------|:----------:|
| 단일 CRUD / 단일 UseCase | 1 |
| 여러 도메인 걸친 기능 | 2~4 |
| 대규모 리팩토링 / 신규 서브시스템 | 5+ (사용자에게 사전 확인) |

메인은 `TaskCreate`로 각 마일스톤을 등록하고, 사용자에게 분할 결과를 출력한다.

```
📋 마일스톤 계획
  M1/3: 카테고리 도메인 모델 + 리포지토리
  M2/3: 카테고리 CRUD UseCase
  M3/3: 카테고리 Controller + 테스트
```

---

## Phase 2 — 마일스톤별 순차 실행

각 마일스톤에 대해 아래 루프를 돈다.

### Step 1. 기술설계 위임 (Agent D)

메인이 `Agent` 도구로 `design-writer` 호출. D가 TDD 필요 여부를 판단해:

- **TDD 필요** → `docs/backend/design/tdd-{slug}.md`에 문서 작성 후 `TDD_CREATED: <경로>` 반환
- **TDD 불필요** (단순 CRUD 등) → `TDD_SKIPPED: <이유>` 반환

### Step 2. 코드 작성 위임 (Agent A)

메인이 `Agent` 도구로 `code-writer` 호출. 프롬프트에는 다음이 포함:

- 마일스톤 제목, 구체적 요구사항·범위, 관련 레이어·도메인
- D가 작성한 TDD 경로 (TDD_CREATED인 경우)
- 반환 포맷 지정

A의 반환:

```
## 완료 요약
### 변경 파일
- /abs/path/File.kt: 설명
### 핵심 설계 결정
### 빌드/테스트 결과
### 불확실한 부분
```

### Step 3. 아키텍처 검토 위임 (Agent B)

메인이 `Agent` 도구로 `architecture-reviewer` 호출. 프롬프트에는:

- 마일스톤 제목
- A가 반환한 **절대 경로 파일 목록**
- A의 핵심 설계 결정 요약

B의 반환은 두 형태 중 하나:

**PASS**
```
PASS
```

**위반 존재**
```yaml
- file: /abs/path/File.kt
  rule: app-module-guidelines.md:Controller 체크리스트 "@Valid 적용"
  line_range: 52-56
  old_string: |
    ...
  new_string: |
    ...
  reason: ...
```

### Step 4. 분기

- `PASS` → 마일스톤 완료 → 다음 마일스톤
- 위반 존재 → Step 5로

### Step 5. 위반 수정 위임 (Agent A 재호출)

메인이 같은 마일스톤의 A 인스턴스를 이어서 사용. B가 반환한 YAML을 프롬프트에 포함하고 지시:

- 각 위반의 `old_string → new_string`을 **정확히** Edit으로 적용
- 임의 해석·확장 금지
- old_string이 매칭되지 않으면 **실패로 보고** (추측 금지)
- 모든 수정 후 컴파일 확인

A의 반환:

```
## 수정 적용 결과
### 적용 성공
### 적용 실패
### 빌드 결과
```

### Step 6. 재검토 (Agent B)

Step 3을 반복. 결과에 따라:
- `PASS` → 다음 마일스톤
- 위반 여전 → Step 5로 돌아가 반복

### Step 7. Escalation (5회 초과 시)

`iter > 5`면 메인이 사용자에게 선택지 제시:

```
⛔ [M{n}] 자동 수렴 실패 — A↔B 루프 5회 초과

선택지:
  1. 현재 상태로 커밋하고 수동 검토
  2. 메인 Claude가 직접 개입하여 수정
  3. 요구사항을 재정의하고 해당 마일스톤 재시작
  4. 해당 마일스톤을 스킵하고 다음으로 진행
  5. 전체 중단
```

> 2번 선택 시에 **한해** 메인이 직접 Edit 허용.

---

## Phase 3 — 완료 보고

모든 마일스톤이 완료되면:

```
✨ 구현 완료

마일스톤: 3/3
총 A↔B 라운드: 5
마일스톤별 라운드: M1=1, M2=3, M3=1
변경 파일 총 12개

다음 단계 제안:
  1. 마일스톤 단위 커밋 작성
  2. 또는 전체를 하나의 커밋으로
```

---

## 왜 이 흐름이 작동하는가

### 설계 먼저, 구현 나중

D가 TDD를 먼저 작성하면 A는 "무엇을 어떻게 짤지"를 미리 확정한 상태에서 코드를 쓴다.
잘못된 방향으로 구현하다 뒤늦게 재작업하는 비용이 줄어든다.

### 자동 수렴

사람이 매 라운드마다 리뷰어 역할을 하지 않아도, 동일한 규칙에 대해 A가 계속 개선하도록 **유도되는 루프**.
규칙이 `docs/`에 명시되어 있어 B의 판단이 세션마다 일관된다.

### 편향 제거

자기 코드의 규칙 위반은 보통 눈에 띄지 않는다. **다른 에이전트 인스턴스가 같은 문서를 보고 판단**하면 자기 편향이 개입하지 않는다.

### 실패를 구조로 흡수

- A가 실수해도 B가 잡는다
- B가 잘못 지적해도 old_string이 매칭되지 않으면 A가 실패 보고
- 5회 초과 시 에이전트가 스스로 판단하지 않고 **사용자에게 escalate**

---

## 커스터마이징

### 루프 횟수를 바꾸고 싶다

`.claude/skills/implement/SKILL.md`의 Step 2-5의 `iter > 5` 기준을 수정.

### 검토 기준에 다른 문서를 추가하고 싶다

`.claude/agents/architecture-reviewer.md`의 **Source of Truth** 섹션에 문서 추가.

### TDD 작성 조건을 바꾸고 싶다

`.claude/agents/design-writer.md`의 "TDD 필요 여부 판단" 섹션에서 필요/불필요 기준을 수정.

### 새 에이전트를 루프에 끼우고 싶다 (예: 보안 전문 검토자)

1. `.claude/agents/security-reviewer.md` 신규 작성 (Read 전용)
2. `.claude/skills/implement/references/security-reviewer-contract.md` 계약 문서 작성
3. `.claude/skills/implement/SKILL.md`의 Phase 2에 Step 추가 (B 이후 보안 검토 → ...)

---

## 참고

- [CLAUDE_SETUP.md](CLAUDE_SETUP.md) — 에이전트·스킬 정의 파일 상세
- [PHILOSOPHY.md](PHILOSOPHY.md) — 이 루프를 만든 설계 사상
- `.claude/skills/implement/SKILL.md` — 실제 스킬 정의 (진실의 원본)
- `.claude/agents/design-writer.md` — Agent D 정의
- `.claude/agents/code-writer.md` — Agent A 정의
- `.claude/agents/architecture-reviewer.md` — Agent B 정의
