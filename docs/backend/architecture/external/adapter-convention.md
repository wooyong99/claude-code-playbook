# Adapter 컨벤션

---

## 핵심 규칙

**Adapter는 Outbound Port만 구현하고, 외부 서비스 예외를 Port Result 타입으로 번역한다.**

Adapter는 `application` 모듈이 정의한 Outbound Port의 유일한 구현체다. 비즈니스 분기·영속성 접근·다른 Adapter 호출을 포함하지 않으며, 외부 시스템과의 경계에서 발생하는 예외를 상태(enum)와 식별 코드로 변환해 상위 계층에 노출한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 클래스 | `{Provider}{Function}Adapter` | `GiftCardPinValidateAdapter`, `GiftCardPurchasePinAdapter` |
| Mock 클래스 | `{Provider}{Function}MockAdapter` | `GiftCardPinValidateMockAdapter` |
| 구현 대상 Port | `{Provider}{Function}Port` | `GiftCardPinValidatePort`, `GiftCardPurchasePinPort` |
| 기능 단위 | Port의 책임과 1:1 매칭 | 하나의 Adapter가 하나의 Port만 구현 |

한 외부 서비스에서 여러 기능을 사용한다면 기능별로 Adapter를 분리한다. 예: 상품권 서비스는 `PinValidate`, `PurchasePin`(소각), `PurchasePayout`(대금 지급) 세 Adapter로 분리.

---

## 구조와 의존성

**규칙: Adapter는 ApiClient와 Port 타입만 의존한다.**

Adapter는 생성자 주입으로 `{Provider}ServiceApiClient`를 받고, Port에 선언된 Request/Result/Status 타입만 반환한다. 도메인 모델·JPA Entity·다른 Port를 참조하지 않는다.

```kotlin
// ✅ Port 타입만 의존
class GiftCardPinValidateAdapter(
    private val apiClient: GiftCardServiceApiClient,
) : GiftCardPinValidatePort {
    override fun validate(request: ValidateRequest): ValidateResult = ...
}
```

```kotlin
// ❌ 도메인 모델·다른 Adapter·Repository를 끌어오지 않는다
class GiftCardPinValidateAdapter(
    private val apiClient: GiftCardServiceApiClient,
    private val memberRepository: MemberJpaRepository, // ❌
    private val payoutAdapter: GiftCardPurchasePayoutAdapter, // ❌
) : GiftCardPinValidatePort { ... }
```

---

## 예외 → Result 변환

**규칙: Provider Exception 계층을 `catch` 순서대로 분기하고, 각 분기마다 Result.status와 code를 명확히 구분한다.**

ApiClient가 던지는 예외는 네 계층(API / HTTP / Network / Parsing)으로 나뉜다. Adapter는 이를 순서대로 catch하여 Port의 Status enum(`VALID` / `INVALID` / `UNKNOWN`, `SUCCEEDED` / `FAILED` / `UNKNOWN`)으로 매핑한다.

| 예외 | Result status | code 패턴 |
|------|---------------|----------|
| `{Provider}ApiException` | 비즈니스적으로 거절된 상태 (`INVALID` / `FAILED`) | `e.code` (외부 messageCode) |
| `{Provider}HttpException` | 결과 불확정 (`UNKNOWN`) | `HTTP_${e.httpStatus}` |
| `{Provider}NetworkException` | 결과 불확정 (`UNKNOWN`) | `NETWORK_ERROR` |
| `{Provider}Exception` (sealed 부모) | 결과 불확정 (`UNKNOWN`) | `EXTERNAL_ERROR` |

```kotlin
// ✅ 계층 순서대로 catch, status/code 명확
override fun validate(request: ValidateRequest): ValidateResult =
    try {
        val response = apiClient.validate(GiftCardValidateRequest(pinNo = request.pinNo))
        ValidateResult(status = ValidateStatus.VALID, pinCode = response.pinNo, ...)
    } catch (e: GiftCardServiceApiException) {
        ValidateResult(status = ValidateStatus.INVALID, code = e.code, message = e.detailMessage ?: e.message)
    } catch (e: GiftCardServiceHttpException) {
        ValidateResult(status = ValidateStatus.UNKNOWN, code = "HTTP_${e.httpStatus}", message = e.message)
    } catch (e: GiftCardServiceNetworkException) {
        ValidateResult(status = ValidateStatus.UNKNOWN, code = "NETWORK_ERROR", message = e.message)
    } catch (e: GiftCardServiceException) {
        ValidateResult(status = ValidateStatus.UNKNOWN, code = "EXTERNAL_ERROR", message = e.message)
    }
```

```kotlin
// ❌ 예외를 그대로 전파하거나 Exception 한 방으로 잡는다
override fun validate(request: ValidateRequest): ValidateResult {
    val response = apiClient.validate(...) // 예외 전파 → 호출 측이 외부 예외 타입에 결합됨
    return ValidateResult(...)
}

override fun validate(request: ValidateRequest): ValidateResult =
    try { ... } catch (e: Exception) {
        ValidateResult(status = ValidateStatus.UNKNOWN, code = "ERROR") // 상태 구분 불가
    }
```

> Provider 예외 계층 설계는 [exception-convention.md](exception-convention.md) "예외 계층 구성" 섹션 참고

---

## 로깅

**규칙: 요청 시작·응답 수신·예외 발생 세 지점에 비즈니스 식별자 중심 로그를 남기고, 민감정보는 마스킹한다.**

ApiClient의 `@ExternalApiLogging`은 HTTP 단위 로그를 남긴다. Adapter는 Port 호출 맥락(요청 식별자, 핵심 파라미터, Status 매핑 결과)을 별도로 남겨 추적성을 확보한다.

```kotlin
// ✅ 단계별 로그 + 민감정보 마스킹
logInfo { "[GIFT-CARD-PIN] 유효성 검증 요청 - pinNo=${request.pinNo.take(4)}****" }
val response = apiClient.validate(...)
logInfo { "[GIFT-CARD-PIN] 유효성 검증 응답 - pinNo=${response.pinNo.take(4)}****, amount=${response.amount}" }
```

```kotlin
// ❌ 전체 핀번호 노출, 로그 태그 부재
logInfo { "핀 검증 요청: ${request.pinNo}, response=$response" }
```

- 로그 태그는 `[PROVIDER-DOMAIN]` 형식의 대괄호 prefix를 사용한다 (`[GIFT-CARD-PIN]`, `[PAY-ORDER]`).
- 예외 catch 블록은 `logWarn`으로 기록한다. `logError`는 시스템 장애가 확실할 때만 사용한다.

---

## DTO 매핑

**규칙: Port 타입 ↔ 외부 DTO 매핑은 Adapter 내부에서 수행하고, Port 타입은 외부 스키마 세부를 노출하지 않는다.**

Adapter 내부에서 ApiClient 호출 직전에 `Request → {Provider}Request` 매핑, 호출 직후에 `{Provider}Response → Result` 매핑을 한다. DTO의 Jackson 어노테이션이나 원시 응답 구조가 Port 시그니처로 새어나가지 않는다.

```kotlin
// ✅ Adapter가 매핑 책임을 갖는다
val response = apiClient.validate(GiftCardValidateRequest(pinNo = request.pinNo))
ValidateResult(
    status = ValidateStatus.VALID,
    pinCode = response.pinNo,
    amount = BigDecimal.valueOf(response.amount),
    validFrom = LocalDate.parse(response.validFrom),
    validTo = LocalDate.parse(response.validTo),
    ...
)
```

복잡한 매핑은 `private fun toXxx()` / `Xxx.toYyy()` 확장 함수로 분리하되, **Adapter 파일 내에 둔다**. 범용 매퍼 클래스를 별도 생성하지 않는다.

> DTO 구조는 [dto-convention.md](dto-convention.md) "스키마 매핑 원칙" 섹션 참고

---

## 판단 기준

| 상황 | 방식 |
|------|------|
| 동일 Provider의 두 기능 | Adapter 2개로 분리 (기능별 Port와 1:1) |
| Provider가 같고 Request/Response만 다른 경우 | Adapter는 분리, ApiClient는 하나로 공유 |
| Mock이 필요한 Adapter | 실 Adapter `@Profile("!local")` + Mock Adapter `@Profile("local")` |
| Mock이 필요 없는 Adapter | 프로파일 제약 없이 기본 빈 등록 |

---

## 금지 사항

- 다른 Adapter·Repository·UseCase를 주입받지 않는다.
- 외부 예외(`HttpClientErrorException`, `ResourceAccessException` 등)를 Adapter에서 직접 catch하지 않는다. ApiClient가 Provider 예외로 감싸 던지므로, Adapter는 Provider 예외만 다룬다.
- Result 타입을 우회하여 Port 시그니처 밖으로 예외를 던지지 않는다.
- 로그에 핀번호·카드번호·비밀번호 등 민감정보를 전체 출력하지 않는다.
- Port 타입에 외부 DTO(`GiftCardValidateResponse` 등)를 직접 노출하지 않는다.

---

## 체크리스트

- [ ] Adapter 이름이 `{Provider}{Function}Adapter` 패턴을 따르는가?
- [ ] 하나의 Outbound Port만 구현하는가?
- [ ] 생성자 의존성이 ApiClient(혹은 동일 Provider의 보조 컴포넌트)로 제한되는가?
- [ ] Provider 예외 4계층을 순서대로 catch하고 각각 다른 status/code로 매핑했는가?
- [ ] 요청/응답 로그에 비즈니스 식별자가 포함되고, 민감정보가 마스킹됐는가?
- [ ] 외부 DTO가 Port 시그니처로 새어나가지 않는가?
- [ ] Mock이 필요한 경우 `@Profile("!local")` / `@Profile("local")`로 대칭 구성됐는가?
