# 예외 처리 컨벤션 (API Layer)

---

## 핵심 규칙

**모든 예외는 `GlobalExceptionHandler`에서 일관 처리하고, `CoreErrorType → HttpStatus` 매핑은 app 모듈의 `ErrorTypeExtension`이 단독 책임을 진다.**

domain 모듈이 Spring에 의존하지 않도록 HTTP 관심사는 app 계층으로 격리된다. Controller는 개별 예외 처리 로직을 두지 않는다. 내부 예외 메시지는 클라이언트에 노출하지 않고 `ErrorCode.message`로 일관된 메시지를 전달한다.

> 도메인 예외 계층(`CoreException`, `CoreErrorType`, `ErrorCode`, `{Domain}ErrorCode`) 상세는 [../../domain/strategies/exception-convention.md](../../domain/strategies/exception-convention.md) 참고

---

## 관련 컴포넌트

- **`GlobalExceptionHandler`** (`:app:*`): 예외 일괄 처리 진입점 (`@RestControllerAdvice`)
- **`ErrorTypeExtension`** (`:app:*`): `CoreErrorType → HttpStatus` 변환
- **`BaseErrorCode`** (`:app:*`): Spring/Jakarta 예외 전용 에러 코드 (`ErrorCode` 구현)
- **`BaseError`** (`:app:*`): 응답 에러 바디 (`code` / `message`)
- **`BaseResponse`** (`:app:*`): `{ data }` / `{ error }` 표준 응답 래퍼
- **`CoreException`** (`:core:domain`): 도메인 예외 베이스 클래스
- **`CoreErrorType`** (`:core:domain`): HTTP 의미 추상화 (Spring 미의존)

---

## 처리 흐름

```
Exception 발생
    ↓
GlobalExceptionHandler
    ├── CoreException        → errorCode.errorType.toHttpStatus() → BaseResponse.error(BaseError(errorCode))
    ├── Spring / Jakarta 예외 → BaseErrorCode.*                    → BaseResponse.error(BaseError(baseErrorCode))
    └── Exception (최상위)    → BaseErrorCode.INTERNAL_ERROR       → BaseResponse.error(...) + ERROR 로깅
```

---

## 예시

### GlobalExceptionHandler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(CoreException::class)
    fun handleCoreException(e: CoreException): ResponseEntity<BaseResponse<Unit>> {
        val httpStatus = e.errorCode.errorType.toHttpStatus()
        if (httpStatus.is5xxServerError) {
            log.error("[domain-exception] ${e.errorCode.code}", e)
        } else {
            log.warn("[domain-exception] ${e.errorCode.code} — ${e.message}")
        }
        return ResponseEntity.status(httpStatus)
            .body(BaseResponse.error(BaseError(e.errorCode)))
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(e: MethodArgumentNotValidException): ResponseEntity<BaseResponse<Unit>> {
        log.warn("[validation] ${e.message}")
        return ResponseEntity.badRequest()
            .body(BaseResponse.error(BaseError(BaseErrorCode.INVALID_INPUT)))
    }

    @ExceptionHandler(Exception::class)
    fun handleGeneric(e: Exception): ResponseEntity<BaseResponse<Unit>> {
        log.error("[unhandled-exception]", e)
        return ResponseEntity.internalServerError()
            .body(BaseResponse.error(BaseError(BaseErrorCode.INTERNAL_ERROR)))
    }
}
```

### ErrorTypeExtension — CoreErrorType → HttpStatus

```kotlin
fun CoreErrorType.toHttpStatus(): HttpStatus = when (this) {
    CoreErrorType.BAD_REQUEST -> HttpStatus.BAD_REQUEST
    CoreErrorType.UNAUTHORIZED -> HttpStatus.UNAUTHORIZED
    CoreErrorType.FORBIDDEN -> HttpStatus.FORBIDDEN
    CoreErrorType.NOT_FOUND -> HttpStatus.NOT_FOUND
    CoreErrorType.CONFLICT -> HttpStatus.CONFLICT
    CoreErrorType.INTERNAL_ERROR -> HttpStatus.INTERNAL_SERVER_ERROR
}
```

### BaseErrorCode — Spring / Jakarta 예외 전용

```kotlin
enum class BaseErrorCode(
    override val code: String,
    override val message: String,
    override val errorType: CoreErrorType,
) : ErrorCode {
    INVALID_INPUT("INVALID_INPUT", "요청 값이 올바르지 않습니다.", CoreErrorType.BAD_REQUEST),
    INVALID_REQUEST_BODY("INVALID_REQUEST_BODY", "요청 본문을 읽을 수 없습니다.", CoreErrorType.BAD_REQUEST),
    MISSING_PARAMETER("MISSING_PARAMETER", "필수 파라미터가 누락됐습니다.", CoreErrorType.BAD_REQUEST),
    METHOD_NOT_ALLOWED("METHOD_NOT_ALLOWED", "허용되지 않는 HTTP 메서드입니다.", CoreErrorType.BAD_REQUEST),
    UNSUPPORTED_MEDIA_TYPE("UNSUPPORTED_MEDIA_TYPE", "지원하지 않는 미디어 타입입니다.", CoreErrorType.BAD_REQUEST),
    NOT_FOUND("NOT_FOUND", "요청한 리소스를 찾을 수 없습니다.", CoreErrorType.NOT_FOUND),
    INTERNAL_ERROR("INTERNAL_ERROR", "서버 내부 오류가 발생했습니다.", CoreErrorType.INTERNAL_ERROR),
}
```

---

## 예외 유형별 매핑

### Spring / Jakarta 예외

- `MethodArgumentNotValidException` → `INVALID_INPUT` (400)
- `HttpMessageNotReadableException` → `INVALID_REQUEST_BODY` (400)
- `MissingServletRequestParameterException` → `MISSING_PARAMETER` (400)
- `HttpRequestMethodNotSupportedException` → `METHOD_NOT_ALLOWED` (405)
- `HttpMediaTypeNotSupportedException` → `UNSUPPORTED_MEDIA_TYPE` (415)
- `NoResourceFoundException` → `NOT_FOUND` (404)
- `Exception` (최상위) → `INTERNAL_ERROR` (500)

### CoreErrorType → HttpStatus

- `BAD_REQUEST` → 400
- `UNAUTHORIZED` → 401
- `FORBIDDEN` → 403
- `NOT_FOUND` → 404
- `CONFLICT` → 409
- `INTERNAL_ERROR` → 500

---

## 로깅 전략

- `CoreException` (4xx) — WARN, Stack Trace 생략
- `CoreException` (5xx) — ERROR, Stack Trace 포함
- Spring / Jakarta 예외 (4xx) — WARN, Stack Trace 생략
- `Exception` (최상위, 5xx) — ERROR, Stack Trace 포함

예상 가능한 클라이언트 오류(4xx)는 스택 트레이스 없이 WARN으로 남겨 운영 로그 소음을 줄이고, 내부 오류(5xx)는 반드시 ERROR + 스택 포함으로 남겨 진단을 돕는다.

---

## 판단 기준

### 언제 새 BaseErrorCode를 추가하는가

- Spring/Jakarta 프레임워크 예외를 새로 매핑할 필요가 있을 때 → `BaseErrorCode`에 추가
- 도메인 규칙 위반 → **금지** — `{Domain}ErrorCode`에 추가 ([../../domain/strategies/exception-convention.md](../../domain/strategies/exception-convention.md) 참고)

### 언제 GlobalExceptionHandler에 새 @ExceptionHandler를 추가하는가

- 기본 `CoreException` / `Exception` 핸들러로 커버 가능 → 추가 불필요
- Spring/Jakarta 특정 예외 타입을 구분해 `BaseErrorCode`로 매핑 필요 → `@ExceptionHandler({특정예외})` 추가
- 도메인 예외 처리 흐름을 바꿔야 함 → 먼저 `CoreErrorType` / `{Domain}ErrorCode` 설계가 올바른지 재검토

---

## 금지 사항

- Controller에서 `try-catch`로 개별 예외를 처리하지 않는다 — `GlobalExceptionHandler`에 일임.
- domain 모듈에서 Spring `HttpStatus`를 참조하지 않는다 — `CoreErrorType`만 사용.
- 내부 예외 메시지(`e.message`)를 그대로 클라이언트 응답에 노출하지 않는다 — `ErrorCode.message`로 일관화.
- 도메인 규칙 위반을 `BaseErrorCode`에 추가하지 않는다 — 반드시 도메인별 `{Domain}ErrorCode`.
- 5xx 예외를 WARN으로 로깅하지 않는다 — 반드시 ERROR + Stack Trace.
- `GlobalExceptionHandler` 외부에 `@ExceptionHandler`를 분산시키지 않는다.

---

## 체크리스트

- [ ] Controller에 개별 `try-catch`나 `@ExceptionHandler`가 없는가?
- [ ] 도메인 규칙 위반을 `CoreException({Domain}ErrorCode.*)`로 던지는가?
- [ ] `CoreErrorType → HttpStatus` 매핑이 `ErrorTypeExtension.toHttpStatus()`를 통하는가?
- [ ] 4xx 예외는 WARN, 5xx 예외는 ERROR + Stack Trace로 로깅하는가?
- [ ] Spring/Jakarta 예외가 `BaseErrorCode`에 적절히 매핑됐는가?
- [ ] 응답이 `BaseResponse.error(BaseError(errorCode))` 형태로 래핑됐는가?
- [ ] domain 모듈이 Spring `HttpStatus`에 의존하지 않는가?
- [ ] 내부 예외 메시지가 클라이언트에 노출되지 않는가? (반드시 `ErrorCode.message` 사용)
