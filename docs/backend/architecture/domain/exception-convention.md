# 예외 처리 컨벤션 (Domain Layer)

---

## 핵심 규칙

**domain 모듈은 프레임워크에 독립적인 예외 계층을 갖는다. HTTP 상태 코드로의 매핑은 표현 계층(app 모듈)의 책임이다.**

Spring `HttpStatus`나 다른 프레임워크 타입을 직접 의존하지 않고, 순수 Kotlin `enum`(`CoreErrorType`)으로 HTTP 의미론을 추상화한다. 비즈니스 규칙 위반은 `CoreException` + 도메인별 `ErrorCode` enum으로 표현한다.

---

## 구성 요소

### CoreErrorType

HTTP 상태 코드의 의미를 순수 Kotlin enum으로 추상화한다. Spring 의존 없음 — domain 모듈에서 자유롭게 사용 가능하다.

```kotlin
enum class CoreErrorType {
    BAD_REQUEST,    // 400 - 잘못된 입력
    UNAUTHORIZED,   // 401 - 인증 필요
    FORBIDDEN,      // 403 - 권한 없음
    NOT_FOUND,      // 404 - 리소스 없음
    CONFLICT,       // 409 - 충돌
    INTERNAL_ERROR, // 500 - 서버 오류
}
```

### ErrorCode (interface)

모든 에러 코드 enum이 구현해야 하는 계약이다. 각 도메인은 이 인터페이스를 구현한 enum으로 에러 코드를 상수화한다.

```kotlin
interface ErrorCode {
    val code: String           // 에러 코드 식별자 (예: "TENANT_SLUG_DUPLICATED")
    val message: String        // 클라이언트 노출 메시지
    val errorType: CoreErrorType
}
```

### 도메인별 ErrorCode enum

에러 코드는 **도메인별 enum**으로 정의한다. 여러 도메인이 공유하는 범용 잡탕 enum은 두지 않는다.

```kotlin
// tenant/TenantErrorCode.kt
enum class TenantErrorCode(
    override val code: String,
    override val message: String,
    override val errorType: CoreErrorType,
) : ErrorCode {
    TENANT_SLUG_DUPLICATED("TENANT_SLUG_DUPLICATED", "이미 사용 중인 식별자입니다.", CoreErrorType.CONFLICT),
    TENANT_STATUS_ILLEGAL_TRANSITION("TENANT_STATUS_ILLEGAL_TRANSITION", "허용되지 않는 테넌트 상태 변경입니다.", CoreErrorType.BAD_REQUEST),
    // ...
}
```

### CoreException

모든 도메인 예외의 기반 클래스다. `ErrorCode` 인터페이스를 받아 GlobalExceptionHandler가 단일 핸들러로 처리할 수 있게 한다.

```kotlin
open class CoreException(
    val errorCode: ErrorCode,
    cause: Throwable? = null,
) : RuntimeException(errorCode.message, cause)
```

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| ErrorCode enum 클래스명 | `{Domain}ErrorCode` | `TenantErrorCode`, `FsNodeErrorCode`, `OrderErrorCode` |
| enum 키 이름 | `UPPER_SNAKE_CASE`, 도메인 prefix 포함 | `TENANT_SLUG_DUPLICATED`, `ORDER_ALREADY_CONFIRMED` |
| `code` 문자열 값 | enum 키 이름과 동일 | `"TENANT_SLUG_DUPLICATED"` |
| 도메인 전용 예외 클래스 (선택) | `{Domain}Exception extends CoreException` | `FsNodeException(errorCode: FsNodeErrorCode, ...)` |

> app 계층은 HTTP/요청 수준의 자체 `BaseErrorCode`를 둔다. 상세는 `docs/backend/architecture/app/api-convention.md` 참고.

---

## 예외 방법 선택 기준

도메인 로직 안에서 오류를 표현할 때 세 가지 방법을 아래 기준으로 선택한다.

| 상황 | 방법 | 던져지는 예외 | 특징 |
|------|------|--------------|------|
| 함수 인자 전제조건 (팩토리 입력값 등) | `require(...)` | `IllegalArgumentException` | 프로그래머 실수 — 클라이언트 메시지 노출 부적합 |
| 객체 상태 전제조건 (상태 전이 불가) | `check(...)` | `IllegalStateException` | 프로그래머 실수 — 내부 불변식 보호 |
| 비즈니스 규칙 위반 (API 응답 필요) | `throw CoreException(errorCode)` | `CoreException` | 클라이언트에 구조화된 에러 메시지/상태 코드로 전달됨 |

```kotlin
// 팩토리 메서드 내 — 입력값 검증
require(name.isNotBlank()) { "상품명은 비어있을 수 없습니다." }

// 행위 메서드 내 — 상태 전제조건
check(status == OrderStatus.PENDING) { "대기 상태의 주문만 확정할 수 있습니다." }

// 비즈니스 규칙 위반 — 핸들링 가능한 도메인 예외
throw CoreException(TenantErrorCode.TENANT_NOT_FOUND)
```

### 언제 새 ErrorCode enum 값을 추가하는가

| 조건 | 방식 |
|------|------|
| 클라이언트가 구분해서 처리해야 하는 새 실패 케이스 | **새 enum 값 추가** (기존 enum에 append) |
| 동일 `errorType`·동일 의미 케이스 | **기존 enum 값 재사용** (중복 enum 만들지 않음) |
| 여러 도메인에 걸친 공통 실패 | 각 도메인의 `{Domain}ErrorCode`에 도메인 관점 값으로 개별 정의 (공통 enum 금지) |

### 언제 도메인 전용 Exception 클래스를 만드는가

| 조건 | 방식 |
|------|------|
| 기본적으로 | **`CoreException`을 직접 throw** (errorCode만 다르게) |
| 도메인 예외에 추가 필드/로직이 필요 | `CoreException`을 상속한 `{Domain}Exception` 작성 |
| 예외 타입으로 구분 캐치가 필요 | 상속 클래스 작성 — 단, `CoreException` 단일 핸들러로 충분하면 불필요 |

---

## 새 도메인 예외 추가 방법

1. 해당 도메인 패키지에 `ErrorCode` 구현체(enum) 정의

```kotlin
enum class FsNodeErrorCode(
    override val code: String,
    override val message: String,
    override val errorType: CoreErrorType,
) : ErrorCode {
    FSNODE_NOT_FOUND("FSNODE_NOT_FOUND", "파일을 찾을 수 없습니다.", CoreErrorType.NOT_FOUND),
    FSNODE_UPLOAD_FAILED("FSNODE_UPLOAD_FAILED", "파일 업로드에 실패했습니다.", CoreErrorType.INTERNAL_ERROR),
}
```

2. (선택) 필요 시 `CoreException`을 상속한 도메인 전용 예외 클래스 정의

```kotlin
class FsNodeException(errorCode: FsNodeErrorCode, cause: Throwable? = null)
    : CoreException(errorCode, cause)
```

3. GlobalExceptionHandler 변경 없이 자동으로 처리됨 (`ErrorCode` 계약 기반)

---

## 의존성 방향

```
domain (ErrorCode 인터페이스, CoreErrorType, {Domain}ErrorCode enum 들, CoreException)
  ↑
application (Flow/UseCase에서 CoreException throw)
  ↑
app (GlobalExceptionHandler: CoreErrorType → HttpStatus 매핑, BaseErrorCode)
```

- domain은 HTTP를 모른다. `CoreErrorType`을 통해 의미만 전달한다.
- app 계층의 `GlobalExceptionHandler`가 `CoreErrorType` → `HttpStatus`를 매핑한다.
- 새 도메인 예외를 추가해도 app 계층 수정이 필요 없다 (단일 핸들러가 `ErrorCode` 계약만 보기 때문).

---

## 금지 사항

- 도메인 계층에서 Spring `HttpStatus` 등 프레임워크 타입을 직접 의존하지 않는다.
- 여러 도메인이 공유하는 범용 "잡탕" `ErrorCode` enum(예: `CommonErrorCode`)을 만들지 않는다 — 도메인별 `{Domain}ErrorCode`로 소속을 명확히 한다.
- `RuntimeException`이나 커스텀 미정의 예외를 **직접 throw 하지 않는다** — `CoreException(errorCode)`로 통일한다.
- 클라이언트에 노출하지 않을 프로그래머 실수(전제조건 위반)까지 `CoreException`으로 래핑하지 않는다 — `require` / `check`로 표현한다.
- `CoreException` 상속 클래스를 필요 없이 남발하지 않는다 — 단일 `CoreException` + `ErrorCode`만으로 대부분 커버된다.

---

## 체크리스트

- [ ] 도메인 계층이 Spring `HttpStatus` 등 프레임워크 타입에 의존하지 않는가?
- [ ] 에러 코드가 도메인별 `{Domain}ErrorCode` enum에 정의됐는가? (범용 잡탕 enum 사용 금지)
- [ ] enum 키 이름과 `code` 문자열 값이 일치하는가?
- [ ] 입력값 전제조건에는 `require`, 상태 전제조건에는 `check`를 사용했는가?
- [ ] 클라이언트에 응답해야 하는 비즈니스 규칙 위반은 `CoreException(errorCode)`로 던지는가?
- [ ] `RuntimeException`·커스텀 미정의 예외를 직접 throw 하지 않는가?
- [ ] 도메인 전용 `{Domain}Exception` 클래스를 정말 필요할 때만 만들었는가? (기본은 `CoreException` 직접 사용)
