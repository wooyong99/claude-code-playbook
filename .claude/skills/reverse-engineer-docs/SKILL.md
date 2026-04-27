---
name: reverse-engineer-docs
description: 기존 코드베이스를 역공학(reverse engineering)하여 docs/backend 형식의 지식 시스템 문서 모음집을 생성하는 스킬. 신규 프로젝트에 합류했을 때, 문서 없는 레거시 코드베이스를 분석할 때, 또는 기존 프로젝트에 Claude 협업 기반을 구축할 때 반드시 사용하라. "기존 프로젝트 문서화해줘", "코드베이스 분석해서 문서 만들어줘", "아키텍처 문서가 없는데", "레거시 코드 이해를 위한 가이드 필요해", "docs/backend 구조로 정리해줘", "프로젝트 온보딩 문서 만들어줘" 등의 요청 시 즉시 이 스킬을 사용하라.
---

# reverse-engineer-docs

기존 코드베이스를 분석하여 `docs/backend/` 지식 시스템 구조에 맞는 문서 모음집을 생성한다.
Claude가 코드베이스의 실제 패턴·관행을 이해하고 일관된 방식으로 협업할 수 있도록 문서 기반을 빠르게 구축하는 것이 목적이다.

---

## 사전 인터뷰

실행 전에 다음을 사용자에게 확인한다 (이미 대화에서 파악된 항목은 생략):

1. **분석 대상 경로**: 백엔드 코드가 위치한 루트 디렉토리 (기본: 현재 작업 디렉토리)
2. **기존 docs/ 여부**: 이미 일부 문서가 있다면 병합할지, 덮어쓸지, 별도 경로에 생성할지
3. **우선순위**: 특정 계층이나 도메인 영역을 먼저 집중 분석할 필요가 있는지

---

## 출력 구조

```
docs/backend/
├── README.md                         # 전체 문서 네비게이션 허브
├── getting-started.md                # 기술 스택 + 개발 환경 + 온보딩 경로
├── architecture/
│   ├── {layer}/
│   │   ├── {layer}-module-guidelines.md   # 계층 개요·원칙
│   │   └── {concept}-convention.md        # 네이밍·구현 패턴·판단 기준
│   └── ...
├── policies/
│   ├── security.md
│   ├── logging.md
│   ├── transaction-and-consistency.md
│   └── concurrency-and-performance.md
└── design/
    ├── README.md                     # TDD 작성 규칙 (이 코드베이스 맞춤)
    └── sample-tdd.md                 # 대표 기능의 TDD 샘플
```

생성 완료 후 `CLAUDE.md`의 문서 맵 섹션을 업데이트한다.

---

## Phase 1: 코드베이스 탐색

Explore 에이전트를 활용하여 다음을 파악한다. 병렬로 실행할 수 있는 항목은 동시에 실행한다.

### 1-A. 기술 스택 파악

- **빌드 도구**: `build.gradle.kts`, `build.gradle`, `pom.xml`, `package.json`, `Cargo.toml` 등
- **멀티모듈 여부**: `settings.gradle(.kts)`, `workspace.json` 등
- **프레임워크**: Spring Boot, Ktor, NestJS, Django, Rails 등
- **ORM/쿼리**: JPA, QueryDSL, Exposed, Prisma, SQLAlchemy 등
- **핵심 의존성**: 인증, 메시지 브로커, 캐시, 외부 API 클라이언트

### 1-B. 모듈/패키지 구조 파악

각 모듈의 역할을 `build.gradle` dependencies와 패키지 구조로 추론한다.

```
# 탐색 예시
find . -name "build.gradle.kts" | head -20
find . -type d -name "kotlin" | head -20
```

### 1-C. 계층 구조 매핑

코드베이스의 실제 구조를 논리 계층에 매핑한다. 아래는 일반적인 패턴이며, 실제 코드 구조를 보고 귀납적으로 파악한다.

| 논리 계층 | 일반적인 위치 패턴 | 핵심 식별자 |
|-----------|-----------------|-----------|
| API 계층 | `controller/`, `api/`, `web/`, `:app` 모듈 | `@RestController`, `Router`, `Handler` |
| 애플리케이션 계층 | `service/`, `usecase/`, `application/`, `:core:application` | `@Service`, `UseCase`, `Command` |
| 도메인 계층 | `domain/`, `:core:domain` | `Entity`, `Aggregate`, `ValueObject`, `Port` |
| 영속성 계층 | `repository/`, `storage/`, `:infra:storage` | `@Repository`, `JpaRepository`, `QueryDSL` |
| 외부 연동 계층 | `client/`, `external/`, `integration/`, `:infra:external` | `FeignClient`, `RestTemplate`, `WebClient` |

**프로젝트가 이 계층 구조를 따르지 않을 수 있다.** 실제 코드를 보고 계층을 파악한 뒤, 그 계층 구조 그대로 문서화한다.

---

## Phase 2: 계층별 심층 분석

각 확인된 계층에 대해 Explore 에이전트로 병렬 분석한다.
분석 체크리스트는 [`references/analysis-checklist.md`](references/analysis-checklist.md)를 참조한다.

### 각 계층에서 수집할 정보

**네이밍 패턴** (실제 클래스명에서 귀납)
- 클래스명 suffix/prefix 패턴
- 메서드명 동사 패턴
- 파일/패키지 구성 방식

**설계 원칙** (코드 구조에서 역추론)
- 클래스 간 의존성 방향
- 책임 분배 방식 (어떤 로직이 어느 계층에 있는가)
- 인터페이스/추상화 경계

**구현 패턴**
- 핵심 어노테이션 사용 패턴
- 공통 베이스 클래스/인터페이스
- 예외 처리 방식

**실제 예시 코드**
- 대표적인 클래스 2~3개 (파일 경로 포함)
- 해당 계층의 전형적인 코드 스니펫

---

## Phase 3: 크로스커팅 정책 분석

다음 4대 정책을 코드에서 역추론한다.

### security.md
- `SecurityConfig`, `WebSecurityConfigurerAdapter`, `@PreAuthorize` 등 인증/인가 구현
- JWT, Session, OAuth 등 인증 방식
- 비밀번호 처리 (`BCryptPasswordEncoder`, `Argon2` 등)
- 민감 정보 처리 패턴 (마스킹, 암호화)

### logging.md
- 로그 프레임워크 (`Logback`, `Log4j2`, `SLF4J`)
- MDC/MDI 사용 패턴 (`MDC.put()`, Filter에서 설정)
- 로그 레벨 관행 (어느 상황에 어느 레벨?)
- 구조화 로그 형식 (JSON vs 텍스트)

### transaction-and-consistency.md
- `@Transactional` 배치 위치 (어느 계층에 주로 선언?)
- 트랜잭션 전파/격리 레벨 사용 패턴
- 이벤트 기반 일관성 (`@TransactionalEventListener`, outbox 패턴)

### concurrency-and-performance.md
- 비관적/낙관적 잠금 (`@Lock`, `@Version`)
- N+1 해결 방식 (`@EntityGraph`, `fetch join`, `batch size`)
- 캐시 전략 (`@Cacheable`, Redis, Caffeine)
- 비동기 처리 (`@Async`, Coroutine, `CompletableFuture`)

---

## Phase 4: 문서 생성

분석 결과를 바탕으로 문서를 작성한다.
각 문서의 형식 템플릿은 [`references/doc-templates.md`](references/doc-templates.md)를 참조한다.

### 핵심 원칙

**실제 코드 우선**: 이상적 패턴이 아니라 실제 코드베이스에서 발견한 패턴만 문서화한다.
**미완성 명시**: 코드만으로 파악이 안 되는 항목은 `> [분석 필요: {이유}]` 로 표시하고 넘어간다.
**예시 코드 필수**: 모든 규칙과 패턴에 실제 파일 경로를 포함한 코드 예시를 첨부한다.
**과도한 추론 금지**: "아마 이런 의도일 것이다"식 추측을 규칙으로 기술하지 않는다.

### architecture/ 문서 작성 스타일

`architecture/` 하위 문서는 아래 4가지 스타일을 반드시 따른다. 자세한 템플릿은 `references/doc-templates.md`의 "architecture/ 문서 작성 스타일" 섹션을 참조한다.

- **판단 기준 제공**: 모든 문서에 "언제 이 컴포넌트를 쓰고 언제 쓰지 않는가"를 명시하는 `## 판단 기준` 섹션이 있어야 한다. 경계가 흐릿한 케이스까지 포함한다.
- **불릿 형식**: 원칙·책임·판단 기준은 불릿(`-`) 목록으로 작성한다. 테이블은 네이밍 패턴 등 구조적 비교가 꼭 필요한 곳에만 제한적으로 사용한다.
- **서술형 문장**: 각 불릿은 레이블이 아니라 완전한 문장이다. "왜 이 계층인가", "위반하면 무슨 일이 생기는가", "어느 계층에 위임해야 하는가"를 문장 안에서 설명한다.
- **선언형 규칙**: "X는 Y다", "A는 B에 위임한다", "C가 D에 들어오면 E가 의미를 잃는다" 형태로 쓴다. "하세요", "합니다" 같은 명령형·경어체를 쓰지 않는다.

### 문서 생성 순서

1. `docs/backend/README.md` — 발견된 계층 구조 기반의 네비게이션 허브
2. `docs/backend/getting-started.md` — 기술 스택, 모듈 지도, 개발 환경 셋업
3. 계층별 `architecture/` 문서 (가이드라인 → 컨벤션 순서)
4. `policies/` 4대 정책 문서
5. `design/README.md` — 이 코드베이스에 맞춘 TDD 작성 규칙
6. `design/sample-tdd.md` — 기존 코드의 대표 기능을 TDD 형식으로 역문서화

### 계층이 많을 때의 우선순위

한 번에 모든 계층을 완성하기 어려울 경우:
1. **도메인 계층** (핵심 비즈니스 개념)
2. **애플리케이션 계층** (조율 방식)
3. 나머지 계층 순으로 진행하고, 미완성 계층은 `README.md`에 "작성 예정" 표시

---

## Phase 5: CLAUDE.md 업데이트

`CLAUDE.md`의 **문서 맵** 섹션을 생성된 문서 경로로 업데이트한다.
파일이 없으면 새로 생성하고, 있으면 문서 맵 부분만 교체한다.

```markdown
## 문서 맵

### 백엔드 — [`docs/backend/`](docs/backend/)

**시작점**: [백엔드 문서 인덱스](docs/backend/README.md) · [온보딩 가이드](docs/backend/getting-started.md)

**레이어별 가이드라인** ([`docs/backend/architecture/`](docs/backend/architecture/))

| 레이어 | 가이드라인 | 상세 컨벤션 |
|--------|-----------|----------|
| {실제 레이어명} | [{파일명}]({경로}) | [{컨벤션명}]({경로}) ... |

**크로스커팅 정책** ([`docs/backend/policies/`](docs/backend/policies/))
- [security](docs/backend/policies/security.md) · [logging](docs/backend/policies/logging.md) · [transaction-and-consistency](docs/backend/policies/transaction-and-consistency.md) · [concurrency-and-performance](docs/backend/policies/concurrency-and-performance.md)

**설계 문서** — [`docs/backend/design/`](docs/backend/design/)
기능·서브시스템의 기술 설계(TDD). 작성 규칙: [design/README.md](docs/backend/design/README.md) · 샘플: [sample-tdd.md](docs/backend/design/sample-tdd.md)

### 작업 원칙
- 작업 전 해당 레이어/영역의 가이드라인·컨벤션 문서를 먼저 참조한다
- 정책·규칙의 출처는 항상 위 문서 맵의 하위 문서이며, CLAUDE.md는 그 맵 역할만 한다
```

---

## 완료 보고

문서 생성이 끝나면 사용자에게 다음을 보고한다:

1. **생성된 문서 목록** (경로 포함)
2. **`[분석 필요]` 항목 목록** — 코드만으로 파악하지 못한 부분 (사용자 확인 필요)
3. **다음 단계 제안** — 어떤 문서를 먼저 검토하면 좋은지
