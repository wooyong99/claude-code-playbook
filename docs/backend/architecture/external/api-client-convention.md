# ApiClient 컨벤션

---

## 핵심 규칙

**ApiClient는 외부 HTTP 호출만 책임지며, HTTP·네트워크 예외를 Provider 전용 예외로 통일해서 던진다.**

ApiClient는 HTTP 클라이언트를 사용해 외부 엔드포인트를 호출하고, 기술 예외(`HttpClientErrorException`, `ResourceAccessException` 등)를 Provider sealed 예외 계층으로 감싼다. 비즈니스 판단·Port 매핑·결과 해석은 수행하지 않는다. Adapter는 이 예외 계층만 이해하면 되므로 상위 계층이 외부 프레임워크 세부에 결합되지 않는다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 클래스 | `{Provider}ApiClient` | `GiftCardApiClient`, `BiscuitLinkApiClient` |
| 예외 변환 헬퍼 | `handleErrors(apiName, call)` (private) | 모든 Provider 동일 |
| 토큰 인증 헬퍼 | `executeWithToken(apiName, call)` (private) | Bearer Token이 필요한 Provider |
| 엔드포인트 상수 | `{FUNCTION}_ENDPOINT` (companion object) | `BURN_ENDPOINT`, `VALIDATE_ENDPOINT` |
| 응답 타입 상수 | `{FUNCTION}_RESPONSE_TYPE` (`ParameterizedTypeReference`) | `BURN_RESPONSE_TYPE` |

> `{Provider}ServiceApiClient` 형식의 `Service` 접미사는 사용하지 않는다.

---

## 메서드 시그니처

**규칙: 메서드 하나가 엔드포인트 하나를 담당하고, 요청·응답 DTO만 입출력으로 노출한다.**

각 API 호출은 별도 메서드로 분리한다. 파라미터는 해당 호출의 Request DTO(또는 Path/Query 변수)만 받고, 반환 타입은 Response DTO 또는 공통 래퍼 타입이다.

```kotlin
// ✅ 메서드-엔드포인트 1:1
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse = ...
fun validate(request: GiftCardValidateRequest): GiftCardValidateResponse = ...
fun verifyAccount(request: BiscuitLinkVerifyAccountRequest): BiscuitLinkApiResponse<BiscuitLinkVerifyAccountResponse> = ...
```

```kotlin
// ❌ 한 메서드에서 여러 엔드포인트 분기
fun call(type: ApiType, request: Any): Any = when (type) {
    BURN -> restClient.post()...
    VALIDATE -> restClient.post()...
}
```

---

## 외부 API 호출 로깅

**규칙: 외부 API를 호출하는 모든 `public` 메서드에 요청·응답·소요시간 로깅을 적용한다.**

로깅에 포함해야 할 정보:
- provider명, API명, HTTP 메서드, 엔드포인트
- 요청 본문 (Body가 없는 경우 명시)
- 응답 결과 및 소요시간

로깅 방식은 프로젝트 환경에 따라 선택한다:
세부 구현 방식(어노테이션 기반, 수동 로깅 등)과 속성 규칙은 [api-client-logging.md](api-client-logging.md)에 프로젝트별로 정의한다.

---

## 예외 변환 (`handleErrors` 패턴)

**규칙: HTTP/네트워크 예외 변환을 단일 `handleErrors` 헬퍼에 집중시키고, 각 API 메서드는 이를 통해 호출한다.**

HTTP 클라이언트가 던지는 예외를 Provider sealed 예외로 번역하는 공통 블록을 `handleErrors(apiName, call)` 형태의 `private` 헬퍼로 두고, 모든 API 메서드가 이를 거치도록 한다. 매 메서드마다 try-catch를 반복하면 누락·불일치가 발생한다.

```kotlin
// ✅ handleErrors가 예외 변환 전담 (GiftCardApiClient 실제 구현)
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse =
    executeWithToken("핀 소각") { token ->
        restClient.post().uri(BURN_ENDPOINT)
            .header(HttpHeaders.AUTHORIZATION, "Bearer $token")
            .body(request)
            .retrieve()
            .body(BURN_RESPONSE_TYPE)
            .extractPayload("핀 소각")
    }

private fun <T> handleErrors(apiName: String, call: () -> T): T =
    try { call() }
    catch (ex: HttpClientErrorException.Unauthorized) {
        throw GiftCardAuthException(rawMessage = "[$apiName] 인증 실패", cause = ex)
    }
    catch (ex: HttpClientErrorException) {
        val errorResult = parseErrorResult(ex.responseBodyAsString)
        throw GiftCardApiException(
            code = errorResult?.messageCode ?: "ERR_${ex.statusCode.value()}",
            rawMessage = errorResult?.message ?: "클라이언트 오류",
            cause = ex,
        )
    }
    catch (ex: HttpServerErrorException) {
        throw GiftCardServerException(httpStatus = ex.statusCode.value(), rawMessage = ex.responseBodyAsString, cause = ex)
    }
    catch (ex: ResourceAccessException) {
        val rawMessage = when {
            ex.cause is SocketTimeoutException -> "API 응답 시간 초과"
            else -> "네트워크 연결 실패: ${ex.message}"
        }
        throw GiftCardNetworkException(rawMessage = rawMessage, cause = ex)
    }
    catch (ex: GiftCardException) { throw ex }   // 이미 변환된 예외는 re-throw
    catch (ex: Exception) {
        throw GiftCardResponseParsingException(rawMessage = "응답 처리 중 오류: ${ex.message}", cause = ex)
    }
```

### 예외 매핑 규칙

| 원본 예외 | 변환 대상 | 조건 |
|----------|----------|------|
| `HttpClientErrorException.Unauthorized` (401) | `{Provider}AuthException` | 토큰 만료/인증 실패 → `executeWithToken`이 재발급 트리거로 사용 |
| `HttpClientErrorException` (4xx, 401 제외) | `{Provider}ApiException` | 응답 본문에서 비즈니스 에러코드(messageCode) 추출 |
| `HttpServerErrorException` (5xx) | `{Provider}ServerException` | httpStatus 보존 |
| `ResourceAccessException` | `{Provider}NetworkException` | `SocketTimeoutException` 구분 메시지 |
| `{Provider}Exception` (이미 변환된 예외) | 그대로 re-throw | 중첩 변환 방지 |
| 그 외 `Exception` | `{Provider}ResponseParsingException` | 역직렬화/응답 구조 오류 |

> Provider 예외 정의는 [exception-convention.md](exception-convention.md) "예외 계층 구성" 섹션 참고

---

## Bearer Token 인증 (`executeWithToken` 패턴)

**규칙: Bearer Token 인증이 필요한 API는 `executeWithToken` 헬퍼를 통해 호출하고, 401 발생 시 자동으로 토큰을 무효화하고 재발급한다.**

`executeWithToken`은 `TokenHolder`에서 토큰을 가져와 인증 헤더와 함께 호출하며, `AuthException` 발생 시 한 번만 재시도한다. 이를 통해 각 API 메서드에서 토큰 관리 로직을 중복 작성하지 않는다.

```kotlin
// ✅ executeWithToken 패턴 (GiftCardApiClient 실제 구현)
private fun <T> executeWithToken(apiName: String, call: (String) -> T): T {
    val issueTokenRequest = GiftCardIssueTokenRequest(clientId = "#cube")
    val token = tokenHolder.getToken { issueToken(issueTokenRequest) }
    return try {
        handleErrors(apiName) { call(token) }
    } catch (ex: GiftCardAuthException) {
        logInfo { "[$apiName] 토큰 만료 감지 - 토큰 무효화 후 재발급" }
        tokenHolder.invalidate()
        val newToken = tokenHolder.getToken { issueToken(issueTokenRequest) }
        handleErrors(apiName) { call(newToken) }
    }
}
```

- 토큰 발급 자체는 `handleErrors`만 사용한다 (토큰이 없으므로 `executeWithToken` 불가).
- 재시도는 1회만 한다. 재시도 후에도 `AuthException`이 발생하면 상위로 전파된다.

---

## TokenHolder

**규칙: Bearer Token의 캐싱·만료 관리는 `{Provider}TokenHolder` 컴포넌트가 담당한다.**

ApiClient가 직접 토큰을 필드로 들고 있으면 동시성 문제가 발생한다. `TokenHolder`는 `ReentrantLock`으로 단일 진입을 보장하고, 만료 버퍼를 두어 실제 만료 직전에 미리 갱신한다.

```kotlin
// ✅ TokenHolder 구조 (GiftCardTokenHolder / BiscuitLinkTokenHolder)
@Component
class GiftCardTokenHolder {
    @Volatile private var token: String? = null
    @Volatile private var expiresAt: Instant = Instant.EPOCH
    private val lock = ReentrantLock()

    fun getToken(issuer: () -> GiftCardIssueTokenResponse): String {
        if (!isExpired()) return requireNotNull(token)
        return lock.withLock {
            if (isExpired()) {
                val response = issuer()
                token = response.accessToken
                expiresAt = Instant.now().plusSeconds(response.expiresIn.toLong() - BUFFER_SECONDS)
            }
            requireNotNull(token)
        }
    }

    fun invalidate() = lock.withLock { expiresAt = Instant.EPOCH }

    private fun isExpired(): Boolean = Instant.now().isAfter(expiresAt)
}
```

- `getToken` 호출 시 이미 유효한 토큰이 있으면 lock 없이 바로 반환한다(빠른 경로).
- `invalidate()`는 `AuthException` catch 후 `executeWithToken`이 호출한다.

---

## 공통 응답 래퍼 처리

**규칙: 외부가 공통 래퍼(`{ result, payload }` 등)를 반환하면 `extractPayload`로 payload를 꺼내고 null을 Parsing 예외로 처리한다.**

```kotlin
// ✅ payload null을 명시적으로 거르고, 도메인 예외로 승격
private fun <T> GiftCardApiResponse<T>?.extractPayload(apiName: String): T =
    this?.payload ?: throw GiftCardResponseParsingException("[$apiName] 응답 payload가 null입니다.")
```

- 응답 전체를 반환하는 메서드(예: `verifyAccount`, `payout`)는 Adapter가 payload 접근을 담당한다.
- `extractPayload`를 사용하지 않고 `?: throw`로 처리하는 것도 동일 패턴이다.

---

## HTTP 클라이언트 사용

**규칙: 단일 Provider는 하나의 HTTP 클라이언트 빈을 `@Qualifier`로 주입받아 재사용한다.**

공통 원칙:
- Provider별로 별도 빈을 구성하고 `@Qualifier`로 주입받는다.
- 메서드 내부에서 클라이언트를 매번 생성하지 않는다.

사용하는 HTTP 클라이언트 유형과 빈 주입 방식은 [api-client-http-client.md](api-client-http-client.md)에 프로젝트별로 정의한다.

---

## 금지 사항

- 메서드마다 try-catch를 반복 작성하지 않는다. 단일 `handleErrors` 헬퍼로 통일한다.
- HTTP 클라이언트를 메서드 내부에서 매번 재생성하지 않는다.
- Spring 예외(`HttpClientErrorException` 등)를 ApiClient 밖으로 전파하지 않는다.
- 엔드포인트 URL을 메서드 내부 문자열 리터럴로 흩뿌리지 않는다. companion object 상수로 모은다.
- 로깅 없이 외부 호출 메서드를 공개하지 않는다.
- 한 메서드 안에서 여러 외부 엔드포인트를 호출하거나 조건 분기로 호출 대상을 바꾸지 않는다.
- 토큰 관리 로직을 ApiClient 필드에서 직접 처리하지 않는다. `TokenHolder`에 위임한다.

---

## 체크리스트

- [ ] 클래스 이름이 `{Provider}ApiClient` 패턴인가? (`Service` 없음)
- [ ] 모든 외부 호출 메서드에 로깅(요청·응답·소요시간)이 적용됐는가?
- [ ] HTTP 클라이언트를 `@Qualifier`로 주입받고, 매 호출마다 재생성하지 않는가?
- [ ] 예외 변환 로직이 단일 `handleErrors` 헬퍼로 집중됐는가?
- [ ] `HttpClientErrorException.Unauthorized` (401)가 `AuthException`으로 먼저 변환되는가?
- [ ] Bearer Token이 필요한 API는 `executeWithToken` + `TokenHolder`를 통해 호출하는가?
- [ ] HTTP 4xx (401 제외) / 5xx / 네트워크 / Parsing 예외가 각기 다른 Provider 예외 타입으로 매핑되는가?
- [ ] 엔드포인트 URL과 `ParameterizedTypeReference`가 companion object 상수로 모여 있는가?
- [ ] 공통 응답 래퍼의 payload null 처리가 `ResponseParsingException`으로 승격되는가?
