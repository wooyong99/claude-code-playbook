# Design Writer — Input / Output Contract

`implement` 스킬이 `design-writer` 서브에이전트와 주고받는 인터페이스 규격.  
에이전트 파일(`design-writer.md`)이 아닌 이 문서가 입출력 포맷의 단일 출처다.

---

## Input

오케스트레이터가 아래 형식으로 프롬프트를 구성해 전달한다.

```
[마일스톤]: {마일스톤 제목}
[요구사항]:
{구체적 범위·목표}

[프로젝트 컨텍스트]:
  - Kotlin + Spring Boot 멀티모듈, 멀티테넌트 e-커머스 플랫폼
  - 의존 방향: app → application → domain ← infra
  - 관련 레이어: {app/application/domain/storage 중 해당}
  - 관련 도메인: {도메인명}

[체크포인트 파일]: .claude/checkpoints/design-writer-{run_id}-M{n}.md

[출력 규격]: 이 문서(.claude/skills/implement/references/design-writer-contract.md) — Output 섹션 그대로.
```

체크포인트 재호출 시 프롬프트에 아래 필드가 추가된다:

```
[체크포인트]: [체크포인트 파일] 경로 참조. 이어서 작업 진행.
```

---

## Output

작업 완료 후 반드시 아래 포맷 중 하나로 반환한다.

### Case A: TDD 작성 완료

출력 **첫 줄**:

```
TDD_CREATED: {docs/backend/design/tdd-{feature-slug}.md 절대 경로}
```

이후 아래 요약을 이어서 작성한다:

```
## 설계 요약

### 핵심 아키텍처 결정
- <결정 1>
- <결정 2>

### 주요 도메인 모델
- <모델명>: <역할 요약>

### 트랜잭션·정합성 전략
- <전략 요약>

### 구현 시 주의사항
- <code-writer에게 전달할 설계 제약·선택>
```

### Case B: TDD 불필요

출력 **단일 행**:

```
TDD_SKIPPED: {이유 — 예: "단일 UseCase 추가로 설계 결정 사항 없음"}
```

### Case C: 컨텍스트 체크포인트

컨텍스트 윈도우가 65% 이상 소모된 경우, 출력 **첫 줄**에 아래 신호를 출력한 뒤 지금까지 완료된 작업을 이어서 작성한다:

```
CONTEXT_CHECKPOINT: {[체크포인트 파일] 경로}
```
