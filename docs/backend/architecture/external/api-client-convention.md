# ApiClient 컨벤션

---

## 핵심 규칙

**ApiClient는 외부 HTTP 호출만 책임지며, HTTP·네트워크 예외를 Provider 전용 예외로 통일해서 던진다.**

ApiClient는 `RestClient`를 사용해 외부 엔드포인트를 호출하고, Spring이 던지는 기술 예외(`HttpClientErrorException`, `ResourceAccessException` 등)를 Provider sealed 예외 계층으로 감싼다. 비즈니스 판단·Port 매핑·결과 해석은 수행하지 않는다. Adapter는 이 예외 계층만 이해하면 되므로 상위 계층이 외부 프레임워크 세부에 결합되지 않는다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 클래스 | `{Provider}ServiceApiClient` 또는 `{Provider}ApiClient` | `GiftCardServiceApiClient`, `PayletterApiClient`, `PayNexusApiClient` |
| 엔드포인트 상수 | `{FUNCTION}_ENDPOINT` (companion object) | `BURN_ENDPOINT`, `VALIDATE_ENDPOINT` |
| 응답 타입 상수 | `{FUNCTION}_RESPONSE_TYPE` (`ParameterizedTypeReference`) | `BURN_RESPONSE_TYPE` |

---

## 메서드 시그니처

**규칙: 메서드 하나가 엔드포인트 하나를 담당하고, 요청·응답 DTO만 입출력으로 노출한다.**

각 API 호출은 별도 메서드로 분리한다. 파라미터는 해당 호출의 Request DTO(또는 Path/Query 변수)만 받고, 반환 타입은 Response DTO다. 다건 분기(`when`으로 여러 엔드포인트 호출)는 허용하지 않는다.

```kotlin
// ✅ 메서드-엔드포인트 1:1
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse = ...
fun getBurnResult(idempotencyKey: String): GiftCardBurnResponse = ...
fun validate(request: GiftCardValidateRequest): GiftCardValidateResponse = ...
```

```kotlin
// ❌ 한 메서드에서 여러 엔드포인트 분기
fun call(type: GiftCardApiType, request: Any): Any = when (type) {
    BURN -> restClient.post()...
    VALIDATE -> restClient.post()...
}
```

---

## `@ExternalApiLogging` 필수 부착

**규칙: 외부 API를 호출하는 모든 `public` 메서드는 `@ExternalApiLogging`을 부착한다.**

AOP가 요청·응답·소요시간을 일관된 포맷으로 기록한다. `requestBody`는 SpEL 표기로 파라미터명을 지정하며, 멀티 테넌트 맥락이 있으면 `tenantId`에도 동일하게 적어준다.

```kotlin
// ✅ 모든 외부 호출 메서드에 부착
@ExternalApiLogging(
    provider = "GIFTCARD_SERVICE",
    apiName = "핀 소각",
    httpMethod = HttpMethod.POST,
    endpoint = BURN_ENDPOINT,
    tenantId = "",
    requestBody = "#request",
)
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse = ...
```

```kotlin
// ❌ 어노테이션 누락 → AOP 로그 유실
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse = ...
```

- `provider`는 대문자 언더스코어(`GIFTCARD_SERVICE`, `PAYLETTER`, `ALIGO`)로 통일한다.
- `tenantId`는 SpEL(`#tenantId`)로 지정하되, Provider가 멀티 테넌트 맥락을 요구하지 않으면 빈 문자열로 둔다.

---

## 예외 변환 (`execute` 패턴)

**규칙: HTTP/네트워크 예외 변환을 단일 `execute` 헬퍼에 집중시키고, 각 API 메서드는 호출만 위임한다.**

Spring `RestClient`가 던지는 예외를 Provider sealed 예외로 번역하는 공통 블록을 `execute(apiName, call)` 형태의 `private` 헬퍼로 두고, 모든 API 메서드가 이를 거치도록 한다. 매 메서드마다 try-catch를 반복하면 누락·불일치가 발생한다.

```kotlin
// ✅ 단일 execute가 예외 변환 전담
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse =
    execute("핀 소각") {
        restClient.post().uri(BURN_ENDPOINT)
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .retrieve()
            .body(BURN_RESPONSE_TYPE)
            .extractPayload("핀 소각")
    }

private fun <T : Any> execute(apiName: String, call: () -> T): T =
    try { call() }
    catch (ex: HttpClientErrorException) {
        val err = parseErrorResult(ex.responseBodyAsString)
        throw GiftCardServiceApiException(
            code = err?.messageCode ?: "ERR_${ex.statusCode.value()}",
            message = err?.message ?: "클라이언트 오류",
            detailMessage = err?.detailMessage,
            cause = ex,
        )
    } catch (ex: HttpServerErrorException) {
        throw GiftCardServiceHttpException(ex.statusCode.value(), "…", ex)
    } catch (ex: ResourceAccessException) {
        throw GiftCardServiceNetworkException("…", ex)
    } catch (ex: GiftCardServiceException) { throw ex }
    catch (ex: Exception) {
        throw GiftCardServiceResponseParsingException("[$apiName] …", ex)
    }
```

```kotlin
// ❌ 메서드마다 try-catch 반복 → 변환 규칙이 어긋날 위험
fun burn(request: GiftCardBurnRequest): GiftCardBurnResponse =
    try {
        restClient.post()...body(BURN_RESPONSE_TYPE)!!
    } catch (ex: HttpClientErrorException) { throw RuntimeException(...) } // 변환 일관성 깨짐
```

### 예외 매핑 규칙

| 원본 예외 | 변환 대상 | 조건 |
|----------|----------|------|
| `HttpClientErrorException` (4xx) | `{Provider}ApiException` | 응답 본문에서 비즈니스 에러코드(messageCode) 추출 |
| `HttpServerErrorException` (5xx) | `{Provider}HttpException` | httpStatus 보존 |
| `ResourceAccessException` | `{Provider}NetworkException` | `SocketTimeoutException` 구분 메시지 |
| `{Provider}Exception` (이미 변환된 예외) | 그대로 re-throw | 중첩 변환 방지 |
| 그 외 `Exception` | `{Provider}ResponseParsingException` | 역직렬화/응답 구조 오류 |

> Provider 예외 정의는 [exception-convention.md](exception-convention.md) "예외 계층 구성" 섹션 참고

---

## 공통 응답 래퍼 처리

**규칙: 외부가 공통 래퍼(`{ result, payload }` 등)를 반환하면 `extractPayload`로 payload를 꺼내고 null을 Parsing 예외로 처리한다.**

```kotlin
// ✅ payload null을 명시적으로 거르고, 도메인 예외로 승격
private fun <T> GiftCardServiceResponse<T>?.extractPayload(apiName: String): T =
    this?.payload ?: throw GiftCardServiceResponseParsingException("[$apiName] 응답 payload가 null입니다.")
```

```kotlin
// ❌ !! 로 NPE 유발 → 상위에서 원인 파악 불가
.body(BURN_RESPONSE_TYPE)!!.payload!!
```

---

## RestClient 사용

**규칙: 단일 Provider는 하나의 `RestClient` 빈을 `@Qualifier`로 주입받아 재사용한다.**

`RestClient`를 메서드 호출 시마다 `RestClient.builder().build()`로 새로 만들지 않는다. Provider 단위로 빈을 생성해 커넥션 풀·타임아웃을 재사용한다.

```kotlin
// ✅ 빈 주입
class GiftCardServiceApiClient(
    @Qualifier("giftCardServiceRestClient") private val restClient: RestClient,
    private val objectMapper: ObjectMapper,
)
```

```kotlin
// ❌ 매 호출마다 빌더
fun burn(...): ... {
    val client = RestClient.builder().baseUrl(...).build() // 새 커넥션
    ...
}
```

> 빈 구성은 [config-convention.md](config-convention.md) "RestClient 빈" 섹션 참고

---

## 로깅

- 메서드 진입·성공 응답은 `@ExternalApiLogging`이 담당하므로 ApiClient 내부에서 `logInfo`를 추가하지 않는다.
- Adapter 계층에서 이미 비즈니스 맥락 로그를 남기므로 ApiClient의 이중 로깅을 피한다.
- `@ExternalApiLogging`이 민감정보를 그대로 기록하지 않도록 `requestBody` SpEL은 DTO 단위로 지정하고, DTO 자체에 마스킹이 필요하면 `toString` 오버라이드를 사용한다.

---

## 금지 사항

- 메서드마다 try-catch를 반복 작성하지 않는다. 단일 `execute` 헬퍼로 통일한다.
- `RestClient`를 메서드 내부에서 `builder().build()`로 재생성하지 않는다.
- Spring 예외(`HttpClientErrorException` 등)를 ApiClient 밖으로 전파하지 않는다.
- 엔드포인트 URL을 메서드 내부 문자열 리터럴로 흩뿌리지 않는다. companion object 상수로 모은다.
- `@ExternalApiLogging` 없이 외부 호출 메서드를 공개하지 않는다.
- 한 메서드 안에서 여러 외부 엔드포인트를 호출하거나 조건 분기로 호출 대상을 바꾸지 않는다.

---

## 체크리스트

- [ ] 클래스 이름이 `{Provider}ServiceApiClient` 또는 `{Provider}ApiClient` 패턴인가?
- [ ] 모든 외부 호출 메서드에 `@ExternalApiLogging`이 부착됐는가?
- [ ] `RestClient`를 `@Qualifier`로 주입받고, 매 호출마다 재생성하지 않는가?
- [ ] 예외 변환 로직이 단일 `execute` 헬퍼로 집중됐는가?
- [ ] HTTP 4xx / 5xx / 네트워크 / Parsing 예외가 각기 다른 Provider 예외 타입으로 매핑되는가?
- [ ] 엔드포인트 URL과 `ParameterizedTypeReference`가 companion object 상수로 모여 있는가?
- [ ] 공통 응답 래퍼의 payload null 처리가 `ResponseParsingException`으로 승격되는가?
