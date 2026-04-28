# Exception 컨벤션

---

## 핵심 규칙

**Provider별로 `sealed class`를 루트로 둔 예외 계층을 선언하고, API / Auth / Server / Network / ResponseParsing 5종을 기본 하위 클래스로 갖는다.**

Adapter는 Provider 예외 타입 하나를 기준으로 `catch` 분기를 구성한다. 계층이 Provider별로 분리되면 다른 Provider의 예외가 섞여 흐를 수 없고, 5종 분류가 통일돼 있으면 Adapter가 Result status 매핑을 일관되게 수행한다. Spring `RestClient`가 던지는 기술 예외는 ApiClient에서 이 계층으로 감싸진다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 파일 | `{Provider}Exception.kt` | `GiftCardException.kt`, `BiscuitLinkException.kt` |
| 루트 | `{Provider}Exception` (sealed) | `GiftCardException`, `BiscuitLinkException` |
| API 예외 | `{Provider}ApiException` | `GiftCardApiException`, `BiscuitLinkApiException` |
| Auth 예외 | `{Provider}AuthException` | `GiftCardAuthException`, `BiscuitLinkAuthException` |
| Server 예외 | `{Provider}ServerException` | `GiftCardServerException`, `BiscuitLinkServerException` |
| Network 예외 | `{Provider}NetworkException` | `GiftCardNetworkException`, `BiscuitLinkNetworkException` |
| Parsing 예외 | `{Provider}ResponseParsingException` | `GiftCardResponseParsingException`, `BiscuitLinkResponseParsingException` |

> `{Provider}` 파트에 `Service`를 포함하지 않는다. `GiftCardServiceException` ❌ → `GiftCardException` ✅

---

## 예외 계층 구성

**규칙: 루트는 `sealed class`이며 `RuntimeException`을 상속하고, 하위 클래스 메시지는 고정 prefix + 가변 정보로 조립한다.**

`sealed`는 Adapter의 순차 `catch`에서 분기 커버리지를 컴파일러가 검증할 수 있게 한다. `open` / 일반 `class`로 두면 제3의 하위 클래스가 추가될 때 Adapter가 놓칠 수 있다.

```kotlin
// ✅ sealed 루트 + 5종 하위 (GiftCard 실제 구현)
sealed class GiftCardException(
    override val message: String,
    override val cause: Throwable? = null,
) : RuntimeException(message, cause)

class GiftCardApiException(
    val code: String,
    val rawMessage: String,
    cause: Throwable? = null,
) : GiftCardException("상품권 서비스 API 오류 [$code]: $rawMessage", cause)

class GiftCardAuthException(
    val rawMessage: String,
    cause: Throwable? = null,
) : GiftCardException("상품권 서비스 인증 오류: $rawMessage", cause)

class GiftCardServerException(
    val httpStatus: Int,
    val rawMessage: String,
    cause: Throwable? = null,
) : GiftCardException("상품권 서비스 서버 오류 [$httpStatus]: $rawMessage", cause)

class GiftCardNetworkException(
    val rawMessage: String,
    cause: Throwable? = null,
) : GiftCardException("상품권 서비스 네트워크 오류: $rawMessage", cause)

class GiftCardResponseParsingException(
    val rawMessage: String,
    cause: Throwable? = null,
) : GiftCardException("상품권 서비스 응답 파싱 실패: $rawMessage", cause)
```

```kotlin
// ❌ open 클래스 + 분류 누락 → Adapter가 모든 경우를 잡을 수 있다는 보장이 없음
open class GiftCardException(message: String) : RuntimeException(message)
class GiftCardApiException(message: String) : GiftCardException(message)
// Auth/Server/Network/Parsing 부재 → Adapter에서 Spring 원시 예외를 직접 catch하게 됨
```

---

## 각 예외의 책임

| 예외 | 발생 시점 | 필수 프로퍼티 | Adapter Result 매핑 |
|------|---------|-------------|--------------------|
| `*ApiException` | 외부 API가 4xx (401 제외)로 비즈니스 거절 | `code: String`, `rawMessage: String` | `INVALID` / `FAILED` + `code = e.code` |
| `*AuthException` | 외부 API가 401 Unauthorized (토큰 만료/불일치) | `rawMessage: String` | ApiClient에서 자동 토큰 재발급에 사용, Adapter까지 전파되지 않음 |
| `*ServerException` | 외부 API가 5xx | `httpStatus: Int`, `rawMessage: String` | `UNKNOWN` / `UNAVAILABLE` + `code = "HTTP_${httpStatus}"` |
| `*NetworkException` | 타임아웃·소켓·DNS 실패 | `rawMessage: String`, `cause` | `UNKNOWN` / `UNAVAILABLE` + `code = "NETWORK_ERROR"` |
| `*ResponseParsingException` | JSON 역직렬화 실패·payload null | `rawMessage: String`, `cause` | `UNKNOWN` + `code = "EXTERNAL_ERROR"` (혹은 별도 분기) |

> `AuthException`은 ApiClient 내부의 `executeWithToken`에서 토큰 재발급 트리거로 사용된다. Adapter까지 전파되지 않는 것이 정상이다.

> Adapter에서의 분기 규칙은 [adapter-convention.md](adapter-convention.md) "예외 → Result 변환" 섹션 참고

---

## 메시지 포맷

**규칙: 메시지는 `{Provider} {분류} [{식별자}]: {rawMessage}` 형식을 고정한다.**

운영 로그·Sentry에서 Provider와 분류를 grep으로 바로 집계할 수 있어야 한다. 메시지에 가변 정보(에러 코드·HTTP status)를 대괄호로 앞세우면 구조적으로 탐색하기 쉽다.

```
상품권 서비스 API 오류 [GIFT_CARD_NOT_FOUND]: 존재하지 않는 상품권입니다
상품권 서비스 인증 오류: [핀 소각] 인증 실패
상품권 서비스 서버 오류 [502]: Bad Gateway
상품권 서비스 네트워크 오류: API 응답 시간 초과
상품권 서비스 응답 파싱 실패: [핀 소각] 응답 payload가 null입니다.
```

---

## `rawMessage` 프로퍼티

**규칙: API 오류 원문(외부 서비스가 반환한 메시지)은 `rawMessage` 프로퍼티로 보존하고, `message` 프로퍼티는 sealed 루트에서 조합된 최종 메시지로 사용한다.**

외부 API가 반환한 원문 메시지를 `rawMessage`로 유지하면 로깅·디버깅 시 Adapter에서 `e.rawMessage`로 원문에 접근할 수 있다.

```kotlin
// ✅ rawMessage 분리
class GiftCardApiException(
    val code: String,
    val rawMessage: String,           // 외부 API 원문 메시지
    cause: Throwable? = null,
) : GiftCardException("상품권 서비스 API 오류 [$code]: $rawMessage", cause)
// e.message → "상품권 서비스 API 오류 [GIFT_CARD_NOT_FOUND]: ..."
// e.rawMessage → "존재하지 않는 상품권입니다" (외부 원문)
```

---

## `cause` 보존

**규칙: Spring/JDK 원시 예외를 Provider 예외로 감쌀 때 반드시 `cause`로 보존한다.**

원인 스택이 없으면 네트워크 장애·역직렬화 버그의 근본 원인을 알 수 없다. ApiClient의 `handleErrors` 헬퍼가 `cause = ex`를 전달하도록 강제한다.

```kotlin
// ✅ 원인 체이닝
catch (ex: HttpClientErrorException) {
    throw GiftCardApiException(code = ..., rawMessage = ..., cause = ex)
}
```

```kotlin
// ❌ 원인 폐기
catch (ex: HttpClientErrorException) {
    throw GiftCardApiException(code = ..., rawMessage = "API 실패") // 스택 추적 불가
}
```

---

## 추가 분류가 필요한 경우

Provider가 특수한 오류 분류를 요구하면 sealed 루트 아래에 하위 클래스로 **추가**한다. 기존 5종을 제거하거나 이름을 바꾸지 않는다.

---

## 금지 사항

- 루트 예외를 `sealed` 외의 `open class` / 일반 `class`로 선언하지 않는다.
- 여러 Provider가 단일 공용 예외 계층(`ExternalApiException` 등)을 공유하지 않는다. Provider 단위로 분리한다.
- `Exception` / `Throwable`을 그대로 상속하지 않는다. `RuntimeException` 기반으로 통일한다.
- 예외 메시지에 민감정보(전체 핀 번호·카드번호·계좌번호)를 포함하지 않는다.
- `cause` 없이 원시 예외를 래핑하지 않는다.
- `{Provider}Service` prefix를 붙이지 않는다 (예: `GiftCardServiceException` ❌).

---

## 체크리스트

- [ ] Provider별로 독립된 `{Provider}Exception.kt` 파일이 존재하는가?
- [ ] 루트 예외가 `sealed class`이고 `RuntimeException`을 상속하는가?
- [ ] API / Auth / Server / Network / ResponseParsing 5종 하위 클래스가 모두 정의됐는가?
- [ ] `ApiException`에 `code: String`과 `rawMessage: String`이 명시적으로 분리돼 있는가?
- [ ] `ServerException`에 `httpStatus: Int`가 있는가?
- [ ] 메시지 포맷이 `{Provider} {분류} [{식별자}]: {rawMessage}` 패턴을 따르는가?
- [ ] 원시 예외를 감쌀 때 `cause`가 보존되는가?
- [ ] 예외 메시지·프로퍼티에 민감정보가 노출되지 않는가?
