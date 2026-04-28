# 레이어별 코드 분석 가이드

각 레이어에서 어떤 파일을 읽고, 어떤 패턴을 찾아야 하는지 설명한다.

---

## Domain 레이어

### 찾아야 할 파일

```bash
# 도메인 클래스
find {path} -name "*.kt" -path "*/domain/*" | grep -v "test"
grep -r "companion object" {path}/domain --include="*.kt" -l
grep -r "CoreException\|DomainException" {path} --include="*.kt" -l
grep -r "ErrorCode\|ErrorType" {path}/domain --include="*.kt" -l
```

### 분석 질문

1. **도메인 객체 생성 방식**: companion object factory? 일반 생성자? data class?
   - `companion object { fun create(...) }` 패턴이 있는가?
   - `reconstitute()` / `of()` / `from()` 패턴이 있는가?

2. **Entity vs Value Object 구분**: id 기반 동등성을 사용하는 클래스가 있는가?
   - `equals/hashCode`를 id로 구현한 클래스 → Entity
   - `data class`로 구현한 클래스 → Value Object

3. **도메인 예외 구조**:
   - 예외 기반 클래스가 있는가? (`CoreException`, `DomainException`)
   - ErrorCode 인터페이스/enum 구조는?
   - HttpStatus를 도메인에서 직접 참조하는가? (안티패턴)

4. **순수성**: Spring, JPA, Jackson annotation이 없는가?

### 핵심 파악 항목

| 항목 | 찾는 패턴 | 전략 결정 |
|-----|---------|---------|
| 객체 생성 | `companion object { fun create }` | Static factory 패턴 사용 여부 |
| 예외 계층 | `class XxxException(val errorCode: ErrorCode)` | 도메인 예외 base 클래스 |
| ErrorCode | `enum class XxxErrorCode : ErrorCode` | 도메인별 ErrorCode enum |
| 행동 메서드 | `fun activate()`, `fun deactivate()` | Rich Domain Model 여부 |

---

## Application 레이어

### 찾아야 할 파일

```bash
# UseCase 클래스
grep -r "@Service" {path}/application --include="*.kt" -l
grep -r "UseCase\|Interactor" {path}/application --include="*.kt" -l

# Flow/Orchestrator
grep -r "Flow\|Orchestrator" {path}/application --include="*.kt" -l

# 각종 컴포넌트
grep -r "Validator\|Handler\|Policy\|EventHandler\|Mapper" {path}/application --include="*.kt" -l
grep -r "@EventListener\|@TransactionalEventListener" {path}/application --include="*.kt" -l

# Port 인터페이스
grep -r "interface.*Port\|interface.*Repository" {path}/application --include="*.kt" -l
```

### 분석 질문

1. **진입점 패턴**: UseCase가 있는가? 직접 Service에 진입하는가?
   - `{Action}{Entity}UseCase` 클래스 존재 여부
   - UseCase 메서드에 `@Transactional`이 있는가 아니면 Flow에 있는가?

2. **오케스트레이션 계층**: UseCase와 Service/Flow 사이 계층이 있는가?
   - UseCase → Flow → Validator/Handler 구조인가?
   - UseCase → Service 단일 구조인가?

3. **포트(Port) 추상화**: application layer에 Port 인터페이스가 있는가?
   - `interface XxxPort` 또는 `interface XxxRepository` (application 패키지에 있을 경우)

4. **이벤트 처리**: `@EventListener` 또는 `@TransactionalEventListener`가 있는가?

5. **매핑**: 도메인 결과를 응용 응답으로 변환하는 별도 클래스가 있는가?

### 핵심 파악 항목

| 항목 | 찾는 패턴 | 전략 결정 |
|-----|---------|---------|
| 진입점 | `class CreateOrderUseCase` | UseCase 패턴 |
| 오케스트레이션 | `class CreateOrderFlow` | Flow 분리 여부 |
| 검증 | `class OrderValidator` | Validator 클래스 |
| 위임 | `class OrderHandler` | Handler/ACL 분리 여부 |
| 전략 | `class OrderPolicy` | Policy 패턴 사용 여부 |
| 이벤트 | `class OrderEventHandler` | 이벤트 처리 방식 |
| 변환 | `class OrderMapper` | 별도 Mapper 여부 |

---

## Storage 레이어

### 찾아야 할 파일

```bash
# JPA Entity
grep -r "@Entity" {path}/storage --include="*.kt" -l
grep -r "@Table" {path}/storage --include="*.kt" -l

# Repository
grep -r "JpaRepository\|CrudRepository" {path}/storage --include="*.kt" -l
grep -r "JPAQueryFactory\|QEntity\|QueryDsl" {path}/storage --include="*.kt" -l
grep -r "JOOQ\|DSLContext" {path}/storage --include="*.kt" -l

# Adapter
grep -r "class.*Adapter" {path}/storage --include="*.kt" -l

# 변환 함수
grep -r "fun.*toDomain\|fun.*toEntity\|fun.*toModel" {path}/storage --include="*.kt" -l
grep -r "extension\|Extension" {path}/storage --include="*.kt" -l
```

### 분석 질문

1. **ORM 선택**: JPA? JOOQ? Exposed? 순수 JDBC?
   - `@Entity` 어노테이션 유무
   - `JPAQueryFactory` 또는 `DSLContext` 유무

2. **QueryDsl 사용 여부**: `QEntity` 클래스나 `JPAQueryFactory` 주입 패턴이 있는가?

3. **Repository 분리**: 단순 CRUD용 JpaRepository와 복잡 쿼리용 Repository가 분리되어 있는가?
   - `XxxJpaRepository : JpaRepository` 패턴
   - `XxxQueryDslRepository` 또는 `XxxCustomRepository` 패턴

4. **Adapter 패턴**: Port를 구현하는 Adapter 클래스가 있는가?
   - `class XxxAdapter : XxxPort`

5. **변환 방식**: 도메인 ↔ 인프라 Entity 변환을 어디서 하는가?
   - Extension 함수 (`fun XxxEntity.toDomain(): Xxx`)
   - 별도 Mapper 클래스
   - Adapter 내부

6. **DDL 관리**: Flyway? Liquibase? 직접 DDL 스크립트?

### 핵심 파악 항목

| 항목 | 찾는 패턴 | 전략 결정 |
|-----|---------|---------|
| ORM | `@Entity`, `JPAQueryFactory` | JPA + QueryDsl |
| 단순 저장소 | `interface XxxJpaRepository : JpaRepository` | JpaRepository |
| 복잡 쿼리 | `class XxxQueryDslRepository` | QueryDsl/JOOQ |
| Port 구현 | `class XxxAdapter : XxxPort` | Adapter 패턴 |
| 변환 | `fun XxxEntity.toDomain()` | Extension 함수 |

---

## External 레이어

### 찾아야 할 파일

```bash
# HTTP 클라이언트
grep -r "WebClient\|RestTemplate\|FeignClient\|HttpClient" {path}/external --include="*.kt" -l

# 외부 API 클라이언트
grep -r "class.*ApiClient\|class.*Client" {path}/external --include="*.kt" -l

# Adapter
grep -r "class.*Adapter" {path}/external --include="*.kt" -l

# 외부 DTO
grep -r "data class.*Request\|data class.*Response" {path}/external --include="*.kt" -l

# 예외 처리
grep -r "WebClientResponseException\|FeignException\|HttpClientError" {path}/external --include="*.kt" -l

# Mock
grep -r "MockAdapter\|StubClient\|FakeClient" {path}/external --include="*.kt" -l
```

### 분석 질문

1. **HTTP 클라이언트**: RestTemplate? WebClient? Feign? OkHttp?

2. **추상화 구조**: ApiClient base 클래스가 있는가?
   - `abstract class BaseApiClient` 또는 `open class ApiClient`

3. **로깅 전략**: API 호출 로깅을 어디서 하는가?
   - Interceptor? Filter? ApiClient base 클래스?
   - 어떤 정보를 로깅하는가? (Request/Response body? 헤더?)

4. **예외 변환**: 외부 예외를 도메인/애플리케이션 예외로 변환하는 패턴이 있는가?

5. **Mock 전략**: 개발/테스트용 Mock 구현이 있는가?

6. **Config 패턴**: 클라이언트 설정을 어떻게 주입하는가? (`@ConfigurationProperties`)

### 핵심 파악 항목

| 항목 | 찾는 패턴 | 전략 결정 |
|-----|---------|---------|
| HTTP 클라이언트 | `WebClient.Builder`, `RestTemplate` | HTTP 클라이언트 종류 |
| Base 클래스 | `abstract class BaseApiClient` | 템플릿 패턴 여부 |
| 로깅 | `ExchangeFilterFunction`, `ClientHttpRequestInterceptor` | 로깅 방식 |
| 예외 변환 | `catch (e: WebClientResponseException)` | 예외 처리 계층 |
| Mock | `class StubXxxApiClient` | Mock 구현 여부 |

---

## App 레이어

App 레이어는 **단일 진입점 모듈**(`{app_module}`)일 수도 있고, 채널·역할별로 **복수의 진입점 모듈**로 분리될 수 있다.
먼저 모듈 구조를 파악한 뒤 분석 경로를 결정한다.

### Step 0. 멀티 모듈 감지

```bash
# 빌드 설정에서 모듈 목록 확인
cat {path}/settings.gradle.kts   # Gradle 프로젝트
cat {path}/pom.xml               # Maven 프로젝트 (modules 태그)

# @SpringBootApplication 위치로 진입점 모듈 파악
grep -r "@SpringBootApplication" {path} --include="*.kt" -l
```

- `@SpringBootApplication`이 **복수** 발견되면 → **멀티 app 모듈** 구조이며, 발견된 모듈명들이 `{app_module}` 목록이 된다
- 하나만 발견되면 → **단일 app 모듈** 구조

멀티 app 모듈인 경우 아래 명령에서 `{app_module}` 자리에 각 모듈명을 순서대로 대입하여 분석하고,
결과를 **하나의 app 레이어**로 통합해 단일 `app/strategies/` 문서를 작성한다.

### 찾아야 할 파일

```bash
# Controller 파일 경로 — 배치 방식(flat vs 서브패키지) 파악
find {path}/{app_module} -name "*Controller.kt" | grep -v "build/" | head -20

# Controller 서브패키지 존재 여부
find {path}/{app_module} -type d -name "controller" | grep -v "build/"

# Global Exception Handler 위치 (어느 모듈에 있는지 확인)
grep -r "@ControllerAdvice\|@RestControllerAdvice" {path} --include="*.kt" -l | grep -v "build/" | grep -v "/test/"

# 공통 응답 클래스 위치 (어느 모듈에 선언됐는가)
grep -r "class.*Response\|data class.*Response" {path} --include="*.kt" -l | grep -v "build/" | grep -v "/test/" | head -10

# 로컬 DTO 존재 여부 ({app_module} 안에 Request/Response 파일이 있는가)
find {path}/{app_module} -name "*Request*.kt" -o -name "*Response*.kt" | grep -v "build/" | grep -v "/test/" | head -20

# dto/ 서브패키지 존재 여부
find {path}/{app_module} -type d -name "dto" | grep -v "build/"

# DTO ↔ Command 변환 클래스 위치 ({app_module}인지 {application_module}인지)
find {path} -name "*DtoExtension*.kt" -o -name "*Extension*.kt" | grep -v "build/" | grep -v "/test/" | head -20

# Multi-tenancy
grep -r "TenantContext\|TenantHolder\|TenantFilter" {path}/{app_module} --include="*.kt" -l | grep -v "build/"

# common/ 서브패키지 존재 여부
find {path}/{app_module} -type d -name "common" | grep -v "build/"
```

> **Security 위치 확인**: `SecurityFilterChain` 클래스가 `{app_module}`에 있는지 별도 `{infra_module}`에 있는지 먼저 파악한다.
> ```bash
> grep -r "SecurityFilterChain" {path} --include="*.kt" -l | grep -v "build/" | grep -v "/test/"
> ```
> `{infra_module}`에 있으면 → Infrastructure 레이어에서 분석 / `{app_module}`에 있으면 → 이 섹션에서 분석

### 분석 질문

1. **인증 방식**: JWT? Session? OAuth? API Key? 무인증?
   - JWT: `JwtAuthenticationFilter`, `JwtTokenProvider` 패턴
   - Session: `HttpSession` 사용
   - OAuth: `OAuth2LoginConfigurer` 또는 Resource Server 설정

2. **응답 형식**: 모든 응답을 래핑하는 공통 클래스가 있는가?
   - `BaseResponse<T>`, `ApiResponse<T>`, `ResponseWrapper<T>`
   - **어느 모듈**에 선언됐는가? (`{app_module}` vs `{application_module}` vs 별도 공통 모듈)

3. **멀티 테넌시**: 테넌트 컨텍스트가 있는가?
   - 헤더 기반? JWT claim 기반?
   - `TenantContextHolder` 또는 ThreadLocal 패턴

4. **글로벌 예외 처리**: `@ControllerAdvice`에서 어떤 예외를 어떻게 처리하는가?
   - HttpStatus 매핑 방식
   - **어느 모듈**에 선언됐는가? (`{app_module}` vs `{application_module}`)

5. **DTO 변환 방식**: Request DTO를 Application Command로 변환하는 방식
   - Extension 함수? 생성자? Builder?
   - 변환 함수가 `{app_module}`에 있는가, `{application_module}`에 있는가?

6. **Controller 배치 방식**: Controller 파일이 도메인 패키지 루트에 flat하게 있는가, `controller/` 서브패키지에 있는가?
   - `find` 결과의 경로에 `/controller/`가 포함되면 → 서브패키지 배치
   - 포함되지 않으면 → flat 배치

7. **로컬 DTO**: `{app_module}` 안에 `{Domain}Requests.kt` / `{Domain}Responses.kt` 등 로컬 DTO 파일이 있는가?
   - 없으면 → Controller가 `{application_module}`의 요청/응답 패키지를 직접 사용하는 것
   - 있으면 → `{app_module}` 전용 표현 DTO를 별도로 유지하는 것

8. **패키지 구조**: `common/` 서브패키지가 **실제로** 존재하는가?
   - `find` 결과로 `common/` 디렉토리가 확인된 경우에만 `common.md` 문서를 작성한다.
   - 존재하지 않으면 기존 `common.md`를 제거 대상으로 표시한다.

### 핵심 파악 항목

| 항목 | 찾는 패턴 | 전략 결정 |
|-----|---------|---------|
| 멀티 앱 모듈 | 복수의 `@SpringBootApplication` | 단일/멀티 app 모듈 구조 |
| Controller 배치 | `*Controller.kt` 파일 경로의 `/controller/` 포함 여부 | flat vs `{domain}/controller/` 서브패키지 |
| 로컬 DTO | `{app_module}` 내 `*Request*.kt`, `*Response*.kt` 존재 여부 | 로컬 DTO vs `{application_module}` DTO 직접 사용 |
| 공통 응답 위치 | 공통 응답 클래스 파일 경로의 모듈명 | `{app_module}` vs `{application_module}` |
| 예외 처리 위치 | `@RestControllerAdvice` 파일 경로의 모듈명 | `{app_module}` vs `{application_module}` |
| 인증 위치 | `SecurityFilterChain` 파일 경로의 모듈명 | `{app_module}` vs `{infra_module}` |
| DTO 변환 위치 | DTO ↔ Command 변환 클래스 파일 경로의 모듈명 | `{app_module}` vs `{application_module}` |
| 멀티 테넌시 | `TenantContextHolder`, 테넌트 식별 헤더 | 멀티 테넌트 방식 |
| common/ 서브패키지 | `find ... -type d -name "common"` | common.md 유지 여부 |

---

## Infrastructure 레이어

이 레이어는 JPA 영속성 어댑터(storage 역할)와 Spring Security/Config 등 인프라 설정이 **같은 모듈**에 함께 위치하는 프로젝트에서 분석한다.
별도 `storage` 모듈 없이 하나의 인프라 모듈이 두 역할을 겸하는 구조가 이에 해당한다.

모듈명은 프로젝트마다 다를 수 있다(`infrastructure`, `infra`, `adapter`, `persistence` 등).
Step 2-2에서 레이어 매핑 결과로 확인된 실제 모듈명을 `{infra_module}`에 대입한다.

### 찾아야 할 파일

```bash
# JPA Entity
grep -r "@Entity" {path}/{infra_module} --include="*.kt" -l
grep -r "@Table" {path}/{infra_module} --include="*.kt" -l

# Repository
grep -r "JpaRepository\|CrudRepository" {path}/{infra_module} --include="*.kt" -l
grep -r "JPAQueryFactory\|QEntity\|QueryDsl\|querydsl" {path}/{infra_module} --include="*.kt" -l

# Adapter (Port 구현체)
grep -r "class.*Adapter" {path}/{infra_module} --include="*.kt" -l

# Security / Auth
grep -r "SecurityFilterChain\|WebSecurityConfigurerAdapter" {path}/{infra_module} --include="*.kt" -l
grep -r "OncePerRequestFilter\|AuthenticationFilter\|JwtFilter" {path}/{infra_module} --include="*.kt" -l
grep -r "JwtTokenProvider\|TokenService\|JwtUtil" {path}/{infra_module} --include="*.kt" -l

# Spring Config
grep -r "@Configuration" {path}/{infra_module} --include="*.kt" -l
grep -r "@ConfigurationProperties" {path}/{infra_module} --include="*.kt" -l

# 변환 함수
grep -r "fun.*toDomain\|fun.*toEntity\|fun.*toModel" {path}/{infra_module} --include="*.kt" -l
grep -r "extension\|Extension" {path}/{infra_module} --include="*.kt" -l
```

### 분석 질문

1. **ORM 선택**: JPA? JOOQ? Exposed? 순수 JDBC?
   - `@Entity` 어노테이션 유무
   - `JPAQueryFactory` 또는 `DSLContext` 유무

2. **QueryDsl 사용 여부**: `QEntity` 클래스나 `JPAQueryFactory` 주입 패턴이 있는가?

3. **Repository 분리**: 단순 CRUD용 JpaRepository와 복잡 쿼리용 Repository가 분리되어 있는가?
   - `XxxJpaRepository : JpaRepository` 패턴
   - `XxxQueryDslRepository` 또는 `XxxCustomRepository` 패턴

4. **Adapter 패턴**: Port를 구현하는 Adapter 클래스가 있는가?
   - `class XxxAdapter : XxxPort`

5. **변환 방식**: 도메인 ↔ 인프라 Entity 변환을 어디서 하는가?
   - Extension 함수 (`fun XxxEntity.toDomain(): Xxx`)
   - 별도 Mapper 클래스
   - Adapter 내부

6. **Security 위치**: Spring Security 설정이 이 모듈에 있는가?
   - `SecurityFilterChain`, JWT 인증 필터, `UserDetailsService`

7. **인증 방식**: JWT? Session? OAuth? API Key?

8. **멀티 테넌시**: JWT claim에서 `tenantId`를 추출하는가?

### 핵심 파악 항목

| 항목 | 찾는 패턴 | 전략 결정 |
|-----|---------|---------|
| ORM | `@Entity`, `JPAQueryFactory` | JPA + QueryDsl |
| 단순 저장소 | `interface XxxJpaRepository : JpaRepository` | JpaRepository |
| 복잡 쿼리 | `class XxxQueryDslRepository` | QueryDsl/JOOQ |
| Port 구현 | `class XxxAdapter : XxxPort` | Adapter 패턴 |
| 변환 | `fun XxxEntity.toDomain()` | Extension 함수 |
| Security | `SecurityFilterChain`, `JwtAuthenticationFilter` | Security 위치 |
| 인증 방식 | `JwtTokenProvider`, `HttpSession` | JWT/Session/OAuth |
