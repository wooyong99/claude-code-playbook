# Backend

멀티테넌트 e-커머스 플랫폼 백엔드 문서. 아키텍처 가이드라인, 정책, 설계 문서로 구성된다.

---

## 디렉토리 구조

```
docs/backend/
├── getting-started.md # 오리엔테이션 (기술 스택, 로컬 실행)
├── architecture/      # 레이어별 모듈 가이드라인 및 컨벤션
├── policies/          # 레이어를 관통하는 규칙·전략 (멀티테넌트, 보안, 로깅, DDL)
└── design/            # 기술 설계 문서 (TDD)
```

**어디로 가야 할지 모르겠다면**

| 작업 | 출발점 |
|------|-------|
| 특정 레이어에 코드 작성 | [`architecture/`](architecture/) |
| 테넌트/보안/로깅/DDL 등 전역 규칙 확인 | [`policies/`](policies/) |
| 새 기능/서브시스템 설계 | [`design/`](design/) |
| 로컬 환경 세팅 | [getting-started.md](getting-started.md) |

---

## 시작하기

- [Getting Started](getting-started.md) — 기술 스택, 로컬 실행, 프로필

---

## 아키텍처

Clean Architecture + DDD 기반 멀티모듈. 의존 방향: `app → application → domain ← infra`.

### 레이어별 가이드라인 및 컨벤션 ([`architecture/`](architecture/))

| 레이어 | 가이드라인 | 상세 컨벤션 |
|--------|-----------|----------|
| `:app:*` | [app-module-guidelines](architecture/app/app-module-guidelines.md) | [api-convention](architecture/app/api-convention.md) · [exception-handling-convention](architecture/app/exception-handling-convention.md) |
| `:core:application` | [application-module-guidelines](architecture/application/application-module-guidelines.md) | [use-case](architecture/application/use-case-convention.md) · [validator](architecture/application/validator-convention.md) · [handler](architecture/application/handler-convention.md) · [flow](architecture/application/flow-convention.md) · [policy](architecture/application/policy-convention.md) · [mapper](architecture/application/mapper-convention.md) · [event-handler](architecture/application/event-handler-convention.md) |
| `:core:domain` | [domain-module-guidelines](architecture/domain/domain-module-guidelines.md) | [domain-model](architecture/domain/domain-model-convention.md) · [exception](architecture/domain/exception-convention.md) |
| `:infra:storage` | [storage-module-guidelines](architecture/storage/storage-module-guidelines.md) | [storage-adapter](architecture/storage/storage-adapter-convention.md) · [querydsl](architecture/storage/querydsl-convention.md) |
| `:infra:external` | [external-module-guidelines](architecture/external/external-module-guidelines.md) | [adapter](architecture/external/adapter-convention.md) · [api-client](architecture/external/api-client-convention.md) · [dto](architecture/external/dto-convention.md) · [exception](architecture/external/exception-convention.md) · [config](architecture/external/config-convention.md) · [mock-adapter](architecture/external/mock-adapter-convention.md) |

---

## 아키텍처 전역 결정 ([`architecture/`](architecture/))

특정 레이어에 국한되지 않으면서, 프로젝트의 **아키텍처 선택**에 의해 도입되는 규칙.

| 문서 | 설명 |
|------|------|
| [multi-tenant](architecture/multi-tenant.md) | 테넌트 격리 전략, 데이터 분류, 채널별 테넌트 식별 방식 |

---

## 크로스커팅 정책 ([`policies/`](policies/))

프로젝트 도메인·아키텍처 선택과 무관하게, 모든 레이어에 걸쳐 일반적으로 적용되는 기술 정책.

| 문서 | 설명 |
|------|------|
| [security](policies/security.md) | 비밀번호 취급, 인증 컨텍스트 분리, 민감 정보 커밋 금지 |
| [logging](policies/logging.md) | 로그 레벨 기준, MDC 키, 민감 데이터 차단 |

---

## 설계 문서 ([`design/`](design/))

기능·서브시스템의 기술 설계 문서(Technical Design Document). 아키텍처 판단 근거, 계층 분리, 트랜잭션 경계, 동시성 등 코드만으로 읽히지 않는 설계 의도를 담는다.

- 작성 규칙·표준 섹션 → [design/README.md](design/README.md)
- 작성 예시 → [design/sample-tdd.md](design/sample-tdd.md)
