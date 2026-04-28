# {프로젝트명}

{프로젝트 한 줄 설명}

---

## 비즈니스 목표

{비즈니스 목표 및 핵심 기능 설명}

### 사용자와 인터페이스

{사용자 유형 · 인터페이스 · 진입점}

---

## 프로젝트 구조

{디렉토리 구조 및 각 디렉토리 역할}

---

## AI 에이전트 공통 작업 지침

### 1) 코딩하기 전에 먼저 생각하라

- 섣불리 가정하지 않는다.
- 헷갈리는 것은 숨기지 않고 질문한다.
- 선택지가 있을 때 각 장단점을 먼저 설명한다.

### 2) 먼저 단순하게 생각하라

- 문제 해결에 필요한 최소한의 코드만 작성·수정한다.
- 미래 요구를 상상해 불필요한 기능을 미리 구현하지 않는다.

### 3) 수술하듯 필요한 부분만 고쳐라

- 변경 범위를 반드시 필요한 부분으로 제한한다.
- 기존 동작을 불필요하게 건드리지 않는다.

### 4) 목표 중심으로 실행하라

- 성공 기준을 먼저 정의한다.
- 기준이 충족될 때까지 검증을 반복한다.

---

## 문서 맵

### 백엔드 — [`docs/backend/`](../docs/backend/)

**시작점**: [백엔드 문서 인덱스](../docs/backend/README.md)

**레이어별 가이드라인** ([`docs/backend/architecture/`](../docs/backend/architecture/))

| 레이어 | 가이드라인 |
|--------|-----------|
| 표현 계층 (Controller, DTO, 예외 처리) | [app-layer-guidelines](../docs/backend/architecture/app/app-layer-guidelines.md) |
| 응용 계층 (UseCase, Flow, Validator) | [application-layer-guidelines](../docs/backend/architecture/application/application-layer-guidelines.md) |
| 도메인 계층 (Entity, Value Object, 도메인 규칙) | [domain-layer-guidelines](../docs/backend/architecture/domain/domain-layer-guidelines.md) |
| 저장소 계층 (DB 어댑터, Repository) | [storage-layer-guidelines](../docs/backend/architecture/storage/storage-layer-guidelines.md) |
| 외부 연동 계층 (외부 API 어댑터, ApiClient) | [external-layer-guidelines](../docs/backend/architecture/external/external-layer-guidelines.md) |

**크로스커팅 정책** — [`docs/backend/policies/`](../docs/backend/policies/README.md)

**설계 문서** — [`docs/backend/design/`](../docs/backend/design/README.md)

### 프론트엔드 — [`docs/frontend/`](../docs/frontend/)

**시작점**: [프론트엔드 문서 인덱스](../docs/frontend/README.md)

**아키텍처** — [`docs/frontend/architecture/`](../docs/frontend/architecture/)
- [frontend-architecture](../docs/frontend/architecture/frontend-architecture.md) · [folder-structure](../docs/frontend/architecture/folder-structure.md) · [state-management](../docs/frontend/architecture/state-management.md)

**컨벤션** — [`docs/frontend/conventions/`](../docs/frontend/conventions/)
- [api](../docs/frontend/conventions/api-conventions.md) · [code](../docs/frontend/conventions/code-conventions.md) · [component](../docs/frontend/conventions/component-conventions.md) · [naming](../docs/frontend/conventions/naming-conventions.md)

**성능** — [`docs/frontend/performance/`](../docs/frontend/performance/)
- [caching-strategy](../docs/frontend/performance/caching-strategy.md) · [list-optimization](../docs/frontend/performance/list-optimization.md) · [rendering-guidelines](../docs/frontend/performance/rendering-guidelines.md)

**UI/UX** — [`docs/frontend/ui-ux/`](../docs/frontend/ui-ux/)
- [ui-principles](../docs/frontend/ui-ux/ui-principles.md) · [ux-guidelines](../docs/frontend/ui-ux/ux-guidelines.md) · [loading-and-feedback](../docs/frontend/ui-ux/loading-and-feedback.md) · [modal-dialog-guidelines](../docs/frontend/ui-ux/modal-dialog-guidelines.md)

---

## 작업 원칙

- 작업 전 해당 레이어/영역의 가이드라인·컨벤션 문서를 먼저 참조한다
- 정책·규칙의 출처는 항상 위 문서 맵을 따라간 **하위 문서**이며, 이 문서는 그 맵 역할만 수행한다

---

## 코드 작업 오케스트레이션

기능 구현·리팩토링은 `implement` 스킬로 수행한다.

- **메인 Claude** = 오케스트레이터 (요구 분석, 마일스톤 분할, 위임, 반복 종료 판단)
- **Agent D** (`design-writer`) = 마일스톤별 기술설계문서(TDD) 작성. `write-tech-design-doc` 스킬 사용
- **Agent A** (`code-writer`) = 코드 작성·수정·테스트
- **Agent B** (`architecture-reviewer`) = `docs/backend/architecture/*` · `docs/backend/policies/*` 준수 검토 (Read/Glob/Grep만 보유, 수정 권한 없음)

각 마일스톤마다 D(설계) → A(구현) → B(검토) 순서로 진행. A↔B 루프를 최대 5회 반복하여 PASS까지 수렴. 초과 시 사용자 escalate.

상세: [`.claude/skills/implement/SKILL.md`](skills/implement/SKILL.md) · [`.claude/agents/design-writer.md`](agents/design-writer.md) · [`.claude/agents/code-writer.md`](agents/code-writer.md) · [`.claude/agents/architecture-reviewer.md`](agents/architecture-reviewer.md)
