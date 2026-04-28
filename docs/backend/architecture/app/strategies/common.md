# App 계층 — `common/` 패키지 상세

---

## 핵심 규칙

**`common/`은 도메인에 의존하지 않는 표현 계층 전역 관심사만 담으며, `config/`·`advice/`·`response/`·`security/`·`logging/`·`validator/` 서브패키지로 책임을 분리한다.**

`common/`은 app 계층 내 모든 `{domain}/` 패키지가 공유하는 횡단 관심사를 담는다. 특정 도메인에 의존하면 도메인 패키지 간 경계가 무너지므로, `common/`의 클래스는 도메인 클래스를 import하지 않는다.

---

## `config/`

| 항목 | 내용 |
|------|------|
| 역할 | Spring MVC·Jackson·CORS·Interceptor 등 표현 계층 설정 |
| 포함 대상 | `WebMvcConfigurer` 구현체, `Jackson2ObjectMapperBuilderCustomizer`, CORS 설정 Bean, `HandlerInterceptor` 등록 |
| 금지 | 비즈니스 로직, 도메인 클래스 import, `@Service` / `@Repository` 선언 |

```
common/config/
├── WebMvcConfig.kt          ← WebMvcConfigurer 구현
├── JacksonConfig.kt         ← ObjectMapper 커스터마이징
└── CorsConfig.kt            ← CORS 허용 도메인·메서드 정의
```

---

## `advice/`

| 항목 | 내용 |
|------|------|
| 역할 | `@RestControllerAdvice`로 전역 예외를 일괄 처리하고 표준 응답으로 변환 |
| 포함 대상 | `GlobalExceptionHandler`, `ErrorTypeExtension` (`CoreErrorType → HttpStatus` 매핑) |
| 금지 | 도메인 예외 처리 로직을 Controller나 UseCase에 분산, `ErrorTypeExtension` 외부에서 HttpStatus 직접 매핑 |

```
common/advice/
├── GlobalExceptionHandler.kt    ← @RestControllerAdvice, 모든 예외 진입점
└── ErrorTypeExtension.kt        ← CoreErrorType → HttpStatus 변환 (단독 책임)
```

`GlobalExceptionHandler`는 다음 순서로 예외를 처리한다:
1. 도메인 예외 (`CoreException`) → `ErrorTypeExtension`으로 HttpStatus 결정
2. Spring 검증 예외 (`MethodArgumentNotValidException`, `ConstraintViolationException`) → 400
3. 그 외 미처리 예외 → 500

---

## `response/`

| 항목 | 내용 |
|------|------|
| 역할 | HTTP 응답의 표준 봉투(envelope) 타입 정의 |
| 포함 대상 | `BaseResponse<T>`, `ErrorResponse`, `PageResponse<T>`, `BaseErrorCode` 인터페이스, `BaseError` sealed class |
| 금지 | 도메인별 응답 타입, 비즈니스 로직, 도메인 클래스 import |

```
common/response/
├── BaseResponse.kt      ← { success, data, error } 봉투
├── ErrorResponse.kt     ← { code, message } 오류 페이로드
├── PageResponse.kt      ← { content, page, size, totalElements } 페이지 봉투
├── BaseErrorCode.kt     ← 에러 코드 인터페이스
└── BaseError.kt         ← sealed class, 에러 분류 계층
```

모든 Controller 응답은 `BaseResponse<T>`로 래핑한다. 오류 응답은 `GlobalExceptionHandler`가 `BaseResponse.error(...)` 팩토리를 통해 생성한다.

---

## `security/`

| 항목 | 내용 |
|------|------|
| 역할 | Spring Security 설정·인증 필터·인가 규칙 |
| 포함 대상 | `SecurityFilterChain` Bean, JWT / 세션 인증 필터, `UserDetailsService` 구현, 인가 규칙(`antMatchers` / `requestMatchers`) |
| 금지 | 비즈니스 권한 검사를 Security 필터에 포함, 도메인 로직 직접 호출 |

```
common/security/
├── SecurityConfig.kt            ← SecurityFilterChain 정의
├── JwtAuthenticationFilter.kt   ← 토큰 파싱·검증·SecurityContext 설정
└── SecurityUserDetailsService.kt← UserDetailsService 구현
```

`TenantContextHolder`에 `tenantId` 주입은 인증 필터 내부에서 수행한다. 필터 종료 시 반드시 `TenantContextHolder.clear()`를 호출한다.

---

## `logging/`

| 항목 | 내용 |
|------|------|
| 역할 | HTTP 요청·응답 로그 기록, MDC(Mapped Diagnostic Context) 설정, 민감 정보 마스킹 |
| 포함 대상 | `OncePerRequestFilter` 구현체(요청·응답 로그), MDC 키(`requestId`, `tenantId`) 주입·제거 로직, 마스킹 유틸 |
| 금지 | 비즈니스 로직, 도메인 클래스 import, 민감 정보(비밀번호·토큰) 원문 기록 |

```
common/logging/
├── HttpLoggingFilter.kt     ← 요청·응답 body/header 로그
├── MdcLoggingFilter.kt      ← requestId·tenantId MDC 주입
└── MaskingUtil.kt           ← 민감 필드 마스킹 함수
```

MDC에 `requestId`와 `tenantId`를 주입하면 UseCase 로그에서 이 값을 자동으로 포함한다. `MdcLoggingFilter`는 요청 시작에 주입하고 `finally` 블록에서 반드시 제거한다.

---

## `validator/`

| 항목 | 내용 |
|------|------|
| 역할 | 커스텀 Bean Validation 어노테이션과 공통 포맷 Validator |
| 포함 대상 | `@Constraint` 어노테이션, `ConstraintValidator<A, T>` 구현체, 공통 포맷 검증(전화번호·사업자번호·날짜 포맷 등) |
| 금지 | 도메인 비즈니스 규칙 검증(도메인 계층 책임), Port 주입 (DB 조회 등) |

```
common/validator/
├── PhoneNumber.kt           ← @PhoneNumber 어노테이션
├── PhoneNumberValidator.kt  ← ConstraintValidator 구현
├── BusinessNumber.kt        ← @BusinessNumber 어노테이션
└── BusinessNumberValidator.kt
```

`validator/`의 Validator는 형식(포맷)만 검증한다. 데이터 존재 여부·중복 여부 등 DB 조회가 필요한 검증은 application 계층 `Validator` 컴포넌트 책임이다.

---

## 금지 사항

- `common/`이 도메인 패키지의 클래스를 import하지 않는다.
- 도메인별 응답 타입·비즈니스 로직을 `common/response/`에 두지 않는다.
- `CoreErrorType → HttpStatus` 매핑을 `ErrorTypeExtension` 외부에서 수행하지 않는다.
- 비즈니스 권한 검사를 `security/` 필터에 포함하지 않는다 — 도메인 로직을 직접 호출하지 않는다.
- DB 조회가 필요한 도메인 규칙 검증을 `validator/`에 두지 않는다 — 형식(포맷) 검증만.
- 민감 정보(비밀번호·토큰 원문)를 `logging/`에서 기록하지 않는다.
- `@Service`·`@Repository`를 `config/`에 선언하지 않는다.

---

## 체크리스트

- [ ] `common/`이 도메인 패키지 클래스를 import하지 않는가?
- [ ] `advice/`에 `GlobalExceptionHandler`와 `ErrorTypeExtension`이 위치하는가?
- [ ] `CoreErrorType → HttpStatus` 매핑이 `ErrorTypeExtension`에만 존재하는가?
- [ ] `response/`에 도메인별 응답 타입이 포함되지 않는가?
- [ ] `security/`에 비즈니스 권한 검사·도메인 로직 직접 호출이 없는가?
- [ ] `validator/`가 형식 검증만 수행하고 DB 조회를 하지 않는가?
- [ ] `logging/`이 민감 정보(비밀번호·토큰)를 원문으로 기록하지 않는가?
