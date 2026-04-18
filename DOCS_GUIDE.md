# DOCS_GUIDE — `docs/` 디렉토리 탐색 가이드

`docs/`는 **에이전트와 사람이 공유하는 Source of Truth**다.
에이전트는 작업 전 해당 레이어의 가이드라인을 읽고, 검토자는 같은 문서로 준수 여부를 판단한다.

---

## 구조

```
docs/
├── backend/              # Kotlin + Spring Boot 서버 문서
│   ├── README.md         # 백엔드 문서 인덱스
│   ├── getting-started.md
│   ├── architecture/     # 레이어별 가이드라인 + 아키텍처 전역 결정
│   │   ├── multi-tenant.md  # 아키텍처 전역 결정 (프로젝트가 멀티테넌트일 때만)
│   │   ├── app/          # Controller, API 규약, 예외 처리
│   │   ├── application/  # UseCase, Validator, Handler, Flow, Policy, Mapper, EventHandler
│   │   ├── domain/       # 도메인 모델, 예외
│   │   ├── storage/      # JPA 어댑터, QueryDSL, DDL 관리
│   │   └── external/     # 외부 API 어댑터, 클라이언트, DTO
│   └── policies/         # 크로스커팅 정책 (도메인·아키텍처 무관, 전역 기술 정책)
│       ├── security.md
│       └── logging.md
│
└── frontend/             # React 프론트엔드 문서
    ├── README.md         # 프론트엔드 문서 인덱스
    ├── architecture/     # FSD 아키텍처, 폴더 구조, 상태 관리
    ├── conventions/      # 코드, 네이밍, 컴포넌트, API
    ├── performance/      # 렌더링, 캐싱, 리스트 최적화
    └── ui-ux/            # UI 원칙, UX 가이드라인, 로딩, 모달
```

---

## 문서는 3가지 등급으로 구분된다

| 등급 | 위치 | 성격 | 검토 에이전트가 참조? |
|------|------|------|:-------------------:|
| **규정** | `architecture/*`, `policies/*` | 반드시 지켜야 할 규칙 (체크리스트 있음) | ✅ 예 |
| **설계** | `design/*` | 특정 기능의 기술 설계 (TDD) | ❌ 아니오 |
| **계획** | `execution-plans/*` | 기능 구현 실행 계획 (샘플) | ❌ 아니오 |

> **중요**: 검토 에이전트(`architecture-reviewer`)는 `architecture/*`와 `policies/*`에만 근거해 판단한다.
> 설계·계획 문서는 *참고자료*일 뿐 *준수 규칙*이 아니다.

---

## 탐색 가이드 — "어디로 가야 하나?"

### 백엔드 작업할 때

| 작업 | 출발점 |
|------|-------|
| Controller·Request/Response DTO 작성 | [app/app-module-guidelines.md](docs/backend/architecture/app/app-module-guidelines.md) |
| UseCase·Validator·Handler 작성 | [application/application-module-guidelines.md](docs/backend/architecture/application/application-module-guidelines.md) |
| 도메인 모델·예외 작성 | [domain/domain-module-guidelines.md](docs/backend/architecture/domain/domain-module-guidelines.md) |
| JPA 어댑터·QueryDSL | [storage/storage-module-guidelines.md](docs/backend/architecture/storage/storage-module-guidelines.md) |
| 외부 API 연동 | [external/external-module-guidelines.md](docs/backend/architecture/external/external-module-guidelines.md) |
| 멀티테넌트 규칙 확인 | [architecture/multi-tenant.md](docs/backend/architecture/multi-tenant.md) |
| DDL 파일 관리 규칙 확인 | [architecture/storage/ddl-management.md](docs/backend/architecture/storage/ddl-management.md) |
| 로깅·보안 등 전역 정책 확인 | [policies/](docs/backend/policies/) |

### 프론트엔드 작업할 때

| 작업 | 출발점 |
|------|-------|
| 폴더 구조·레이어 의존성 | [architecture/frontend-architecture.md](docs/frontend/architecture/frontend-architecture.md) |
| 상태 관리 | [architecture/state-management.md](docs/frontend/architecture/state-management.md) |
| 네이밍·코드 스타일 | [conventions/](docs/frontend/conventions/) |
| 성능 최적화 | [performance/](docs/frontend/performance/) |
| 로딩·모달·UX 기준 | [ui-ux/](docs/frontend/ui-ux/) |

---

## 각 가이드라인 문서의 공통 구조

`architecture/*/[*-module-guidelines].md` 문서는 아래 구조를 따른다:

```
1. 모듈 목적
2. Coding Rules
3. Naming Rules
4. File Structure
5. Post-Work Verification 체크리스트   ← 검토 에이전트의 핵심 근거
```

**Post-Work Verification 체크리스트**는 에이전트 검토의 기준점이다.
새 규칙을 추가할 때는 반드시 체크리스트에도 반영할 것.

---

## 문서를 수정·추가할 때

### 기존 규칙을 바꿀 때

1. 해당 문서에서 규칙을 수정
2. 같은 문서의 **Post-Work Verification 체크리스트**도 함께 갱신
3. 변경된 규칙의 이유를 commit 메시지나 문서 내부에 기록

### 새 컨벤션 문서를 추가할 때

1. 적절한 레이어 디렉토리 하위에 `*-convention.md` 추가
2. 같은 레이어의 `*-module-guidelines.md`에서 새 문서를 참조하도록 링크
3. `.claude/agents/code-writer.md`의 **Mandatory References** 목록에 등록
4. `.claude/agents/architecture-reviewer.md`의 **Source of Truth** 목록에 등록
5. `.claude/CLAUDE.md`의 문서 맵 테이블에 추가

### 새 정책을 추가할 때

1. `docs/backend/policies/<정책명>.md` 생성
2. 위 3~5번 동일하게 업데이트

> ⚠️ **가장 중요**: 문서를 추가하고도 에이전트 정의에 반영하지 않으면 에이전트는 그 문서를 **읽지 않는다**. 규칙이 없는 것과 같아진다.

---

## 참고

- [PHILOSOPHY.md](PHILOSOPHY.md) — 왜 문서를 Source of Truth로 쓰는가
- [CLAUDE_SETUP.md](CLAUDE_SETUP.md) — 에이전트가 문서를 참조하는 메커니즘
