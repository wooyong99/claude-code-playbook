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

## 문서 맵

### 백엔드 — [`docs/backend/`](../docs/backend/)

**시작점**: [백엔드 문서 인덱스](../docs/backend/README.md)

**레이어별 가이드라인** ([`docs/backend/architecture/`](../docs/backend/architecture/))

| 레이어 | 가이드라인 | 상세 컨벤션 |
|--------|-----------|----------|
| `:app:*` | [app-module-guidelines](../docs/backend/architecture/app/app-module-guidelines.md) | [api](../docs/backend/architecture/app/api-convention.md) · [exception-handling](../docs/backend/architecture/app/exception-handling-convention.md) |
| `:core:application` | [application-module-guidelines](../docs/backend/architecture/application/application-module-guidelines.md) | [use-case](../docs/backend/architecture/application/use-case-convention.md) · [validator](../docs/backend/architecture/application/validator-convention.md) · [handler](../docs/backend/architecture/application/handler-convention.md) · [flow](../docs/backend/architecture/application/flow-convention.md) · [policy](../docs/backend/architecture/application/policy-convention.md) · [mapper](../docs/backend/architecture/application/mapper-convention.md) · [event-handler](../docs/backend/architecture/application/event-handler-convention.md) |
| `:core:domain` | [domain-module-guidelines](../docs/backend/architecture/domain/domain-module-guidelines.md) | [domain-model](../docs/backend/architecture/domain/domain-model-convention.md) · [exception](../docs/backend/architecture/domain/exception-convention.md) |
| `:infra:storage` | [storage-module-guidelines](../docs/backend/architecture/storage/storage-module-guidelines.md) | [storage-adapter](../docs/backend/architecture/storage/storage-adapter-convention.md) · [querydsl](../docs/backend/architecture/storage/querydsl-convention.md) · [ddl-management](../docs/backend/architecture/storage/ddl-management.md) |
| `:infra:external` | [external-module-guidelines](../docs/backend/architecture/external/external-module-guidelines.md) | [adapter](../docs/backend/architecture/external/adapter-convention.md) · [api-client](../docs/backend/architecture/external/api-client-convention.md) · [dto](../docs/backend/architecture/external/dto-convention.md) · [exception](../docs/backend/architecture/external/exception-convention.md) · [config](../docs/backend/architecture/external/config-convention.md) · [mock-adapter](../docs/backend/architecture/external/mock-adapter-convention.md) |

**크로스커팅 정책** ([`docs/backend/policies/`](../docs/backend/policies/))
- [security](../docs/backend/policies/security.md) · [logging](../docs/backend/policies/logging.md)

**설계 문서** — [`docs/backend/design/`](../docs/backend/design/)
기능·서브시스템의 기술 설계(TDD). 작성 규칙과 템플릿은 [design/README.md](../docs/backend/design/README.md) · 샘플 [sample-tdd.md](../docs/backend/design/sample-tdd.md).

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
- **Agent A** (`code-writer`) = 코드 작성·수정·테스트
- **Agent B** (`architecture-reviewer`) = `docs/backend/architecture/*` · `docs/backend/policies/*` 준수 검토 (Read/Glob/Grep만 보유, 수정 권한 없음)

각 마일스톤마다 A→B 루프를 최대 5회 반복하여 PASS까지 수렴. 초과 시 사용자 escalate.

상세: [`.claude/skills/implement/SKILL.md`](skills/implement/SKILL.md) · [`.claude/agents/code-writer.md`](agents/code-writer.md) · [`.claude/agents/architecture-reviewer.md`](agents/architecture-reviewer.md)
