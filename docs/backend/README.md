# Backend

{프로젝트명} 백엔드 문서. 아키텍처 가이드라인, 정책, 설계 문서로 구성된다.

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

| 레이어 | 가이드라인 |
|--------|-----------|
| `:app:*` | [app-module-guidelines](architecture/app/app-module-guidelines.md) |
| `:core:application` | [application-module-guidelines](architecture/application/application-module-guidelines.md) |
| `:core:domain` | [domain-module-guidelines](architecture/domain/domain-module-guidelines.md) |
| `:infra:storage` | [storage-module-guidelines](architecture/storage/storage-module-guidelines.md) |
| `:infra:external` | [external-module-guidelines](architecture/external/external-module-guidelines.md) |

---

## 크로스커팅 정책 ([`policies/`](policies/))

프로젝트 도메인·아키텍처 선택과 무관하게, 모든 레이어에 걸쳐 일반적으로 적용되는 기술 정책.

| 문서 | 설명 |
|------|------|
| [security](policies/security.md) | 비밀번호 취급, 인증 컨텍스트 분리, 민감 정보 커밋 금지 |
| [logging](policies/logging.md) | 로그 레벨 기준, MDC 키, 민감 데이터 차단 |
| [transaction-and-consistency](policies/transaction-and-consistency.md) | 트랜잭션 경계 설정, 정합성 수준 선택, 이벤트 기반 최종 일관성, 분산 락 |
| [concurrency-and-performance](policies/concurrency-and-performance.md) | 동시성 제어 방식 선택, N+1 해결 패턴, 캐시 전략, 확장 가능성 문서화 |

---

## 설계 문서 ([`design/`](design/))

기능·서브시스템의 기술 설계 문서(Technical Design Document). 아키텍처 판단 근거, 계층 분리, 트랜잭션 경계, 동시성 등 코드만으로 읽히지 않는 설계 의도를 담는다.

- 작성 규칙·표준 섹션 → [design/README.md](design/README.md)
- 작성 예시 → [design/sample-tdd.md](design/sample-tdd.md)
