# Exception 컨벤션

---

## 핵심 규칙

**Provider별로 `sealed class`를 루트로 둔 예외 계층을 선언하고, API / HTTP / Network / ResponseParsing 4종을 기본 하위 클래스로 갖는다.**

Adapter는 Provider 예외 타입 하나를 기준으로 `catch` 분기를 구성한다. 계층이 Provider별로 분리되면 다른 Provider의 예외가 섞여 흐를 수 없고, 4종 분류가 통일돼 있으면 Adapter가 Result status 매핑을 일관되게 수행한다. Spring `RestClient`가 던지는 기술 예외는 ApiClient에서 이 계층으로 감싸진다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 파일 | `{Provider}ServiceException.kt` 또는 `{Provider}Exception.kt` | `GiftCardServiceException.kt`, `AligoAlimtalkException.kt` |
| 루트 | `{Provider}ServiceException` (sealed) | `GiftCardServiceException` |
| API 예외 | `{Provider}ServiceApiException` | `GiftCardServiceApiException` |
| HTTP 예외 | `{Provider}ServiceHttpException` | `GiftCardServiceHttpException` |
| Network 예외 | `{Provider}ServiceNetworkException` | `GiftCardServiceNetworkException` |
| Parsing 예외 | `{Provider}ServiceResponseParsingException` | `GiftCardServiceResponseParsingException` |

---

## 예외 계층 구성

**규칙: 루트는 `sealed class`이며 `RuntimeException`을 상속하고, 하위 클래스 메시지는 고정 prefix + 가변 정보로 조립한다.**

`sealed`는 Adapter의 `when` 또는 순차 `catch`에서 분기 커버리지를 컴파일러가 검증할 수 있게 한다. `open` / 일반 `class`로 두면 제3의 하위 클래스가 추가될 때 Adapter가 놓칠 수 있다.

```kotlin
// ✅ sealed 루트 + 4종 하위
sealed class GiftCardServiceException(
    override val message: String,
    override val cause: Throwable? = null,
) : RuntimeException(message, cause)

class GiftCardServiceApiException(
    val code: String,
    message: String,
    val detailMessage: String? = null,
    cause: Throwable? = null,
) : GiftCardServiceException("상품권 서비스 API 에러 [$code]: $message", cause)

class GiftCardServiceHttpException(
    val httpStatus: Int,
    message: String,
    cause: Throwable? = null,
) : GiftCardServiceException("상품권 서비스 HTTP 오류 [$httpStatus]: $message", cause)

class GiftCardServiceNetworkException(
    message: String,
    cause: Throwable? = null,
) : GiftCardServiceException("상품권 서비스 네트워크 오류: $message", cause)

class GiftCardServiceResponseParsingException(
    message: String,
    cause: Throwable? = null,
) : GiftCardServiceException("상품권 서비스 응답 파싱 실패: $message", cause)
```

```kotlin
// ❌ open 클래스 + 분류 누락 → Adapter가 모든 경우를 잡을 수 있다는 보장이 없음
open class GiftCardException(message: String) : RuntimeException(message)
class GiftCardApiException(message: String) : GiftCardException(message)
// Http/Network/Parsing 부재 → Adapter에서 Spring 원시 예외를 직접 catch하게 됨
```

---

## 각 예외의 책임

| 예외 | 발생 시점 | 필수 프로퍼티 | Adapter Result 매핑 |
|------|---------|-------------|--------------------|
| `*ApiException` | 외부 API가 4xx로 비즈니스 거절 | `code: String`, `message`, `detailMessage: String?` | `INVALID` / `FAILED` + `code = e.code` |
| `*HttpException` | 외부 API가 5xx | `httpStatus: Int`, `message` | `UNKNOWN` + `code = "HTTP_${httpStatus}"` |
| `*NetworkException` | 타임아웃·소켓·DNS 실패 | `message`, `cause` | `UNKNOWN` + `code = "NETWORK_ERROR"` |
| `*ResponseParsingException` | JSON 역직렬화 실패·payload null | `message`, `cause` | `UNKNOWN` + `code = "EXTERNAL_ERROR"` (혹은 별도 분기) |

> Adapter에서의 분기 규칙은 [adapter-convention.md](adapter-convention.md) "예외 → Result 변환" 섹션 참고

---

## 메시지 포맷

**규칙: 메시지는 `{Provider} {분류} [{식별자}]: {원인}` 형식을 고정한다.**

운영 로그·Sentry에서 Provider와 분류를 grep으로 바로 집계할 수 있어야 한다. 메시지에 가변 정보(에러 코드·HTTP status)를 대괄호로 앞세우면 구조적으로 탐색하기 쉽다.

```
상품권 서비스 API 에러 [PIN_INVALID]: 유효하지 않은 핀번호입니다
상품권 서비스 HTTP 오류 [502]: 상품권 서비스 서버 오류: <body>
상품권 서비스 네트워크 오류: 상품권 서비스 API 응답 시간 초과
상품권 서비스 응답 파싱 실패: [핀 소각] 응답 payload가 null입니다.
```

---

## `cause` 보존

**규칙: Spring/JDK 원시 예외를 Provider 예외로 감쌀 때 반드시 `cause`로 보존한다.**

원인 스택이 없으면 네트워크 장애·역직렬화 버그의 근본 원인을 알 수 없다. ApiClient의 `execute` 헬퍼가 `cause = ex`를 전달하도록 강제한다.

```kotlin
// ✅ 원인 체이닝
catch (ex: HttpClientErrorException) {
    throw GiftCardServiceApiException(code = ..., message = ..., cause = ex)
}
```

```kotlin
// ❌ 원인 폐기
catch (ex: HttpClientErrorException) {
    throw GiftCardServiceApiException(code = ..., message = "API 실패") // 스택 추적 불가
}
```

---

## 추가 분류가 필요한 경우

Provider가 특수한 오류 분류(예: `NotSupportedPgException` / `InvalidSignatureException`)를 요구하면 sealed 루트 아래에 하위 클래스로 **추가**한다. 기존 4종을 제거하거나 이름을 바꾸지 않는다. Adapter의 기본 분기 코드가 깨지지 않도록 호환을 유지한다.

```kotlin
// ✅ 기존 4종 + 특수 예외 추가
class GiftCardServiceDuplicateRequestException(
    val duplicateRequestId: String,
    cause: Throwable? = null,
) : GiftCardServiceException("상품권 서비스 중복 요청: $duplicateRequestId", cause)
```

---

## 금지 사항

- 루트 예외를 `sealed` 외의 `open class` / 일반 `class`로 선언하지 않는다.
- 여러 Provider가 단일 공용 예외 계층(`ExternalApiException` 등)을 공유하지 않는다. Provider 단위로 분리한다.
- `Exception` / `Throwable`을 그대로 상속하지 않는다. `RuntimeException` 기반으로 통일한다.
- 예외 메시지에 민감정보(전체 핀 번호·카드번호)를 포함하지 않는다.
- `cause` 없이 원시 예외를 래핑하지 않는다.

---

## 체크리스트

- [ ] Provider별로 독립된 `{Provider}Exception.kt` 파일이 존재하는가?
- [ ] 루트 예외가 `sealed class`이고 `RuntimeException`을 상속하는가?
- [ ] API / HTTP / Network / ResponseParsing 4종 하위 클래스가 모두 정의됐는가?
- [ ] 각 예외 클래스가 분류 판단에 필요한 프로퍼티(code, httpStatus 등)를 명시적으로 노출하는가?
- [ ] 메시지 포맷이 `{Provider} {분류} [{식별자}]: {원인}` 패턴을 따르는가?
- [ ] 원시 예외를 감쌀 때 `cause`가 보존되는가?
- [ ] 예외 메시지·프로퍼티에 민감정보가 노출되지 않는가?
