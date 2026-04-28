# Adapter 컨벤션

---

## 핵심 규칙

**Adapter는 Outbound Port만 구현하고, 외부 서비스 예외를 Port Result 타입으로 번역한다.**

Adapter는 `application` 모듈이 정의한 Outbound Port의 유일한 구현체다. 비즈니스 분기·영속성 접근·다른 Adapter 호출을 포함하지 않으며, 외부 시스템과의 경계에서 발생하는 예외를 상태(enum)와 식별 코드로 변환해 상위 계층에 노출한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 클래스 | `{Provider}{Function}Adapter` | `GiftCardPinValidateAdapter`, `GiftCardPurchaseBurnAdapter`, `BiscuitLinkPayoutAdapter` |
| Mock 클래스 | `Mock{Function}Adapter` | `MockBurnAdapter`, `MockPinValidateAdapter`, `MockPayoutAdapter` |
| 구현 대상 Port | `{Provider}{Function}Port` | `GiftCardPinValidatePort`, `GiftCardPurchaseBurnPort` |
| 기능 단위 | Port의 책임과 1:1 매칭 | 하나의 Adapter가 하나의 Port만 구현 |

> Mock 클래스에서는 Provider prefix가 생략될 수 있다. Mock에서는 `Mock{Function}Adapter` 형식이 더 일반적으로 쓰인다.

한 외부 서비스에서 여러 기능을 사용한다면 기능별로 Adapter를 분리한다. 예: 상품권 서비스는 `PinValidate`, `PurchaseBurn`(소각) Adapter로, BiscuitLink는 `AccountVerification`, `Payout` Adapter로 분리.

---

## 구조와 의존성

**규칙: Adapter는 ApiClient와 Port 타입만 의존한다.**

Adapter는 생성자 주입으로 `{Provider}ApiClient`를 받고, Port에 선언된 Request/Result/Status 타입만 반환한다. 도메인 모델·JPA Entity·다른 Port를 참조하지 않는다.

```kotlin
// ✅ Port 타입만 의존 (GiftCardPinValidateAdapter 실제 구현)
@Component
@Profile("!local")
class GiftCardPinValidateAdapter(
    private val apiClient: GiftCardApiClient,
) : GiftCardPinValidatePort {
    override fun validate(request: ValidateRequest): ValidateResult = ...
}
```

```kotlin
// ❌ 도메인 모델·다른 Adapter·Repository를 끌어오지 않는다
class GiftCardPinValidateAdapter(
    private val apiClient: GiftCardApiClient,
    private val memberRepository: MemberJpaRepository, // ❌
    private val payoutAdapter: BiscuitLinkPayoutAdapter, // ❌
) : GiftCardPinValidatePort { ... }
```

---

## 예외 → Result 변환

**규칙: Provider Exception 계층을 `catch` 순서대로 분기하고, 각 분기마다 Result.status와 code를 명확히 구분한다.**

ApiClient가 던지는 예외는 5계층(API / Auth / Server / Network / ResponseParsing)으로 나뉜다. Adapter는 이를 순서대로 catch하여 Port의 Status enum으로 매핑한다. `AuthException`은 ApiClient 내부 `executeWithToken`에서 처리되므로 Adapter 레벨에서는 일반적으로 나타나지 않는다.

| 예외 | Result status | code 패턴 |
|------|---------------|----------|
| `{Provider}ApiException` | 비즈니스적으로 거절 (`INVALID` / `FAILED`) | `e.code` (외부 messageCode) |
| `{Provider}ServerException` | 결과 불확정 또는 서비스 불가 (`UNKNOWN` / `UNAVAILABLE`) | `HTTP_${e.httpStatus}` |
| `{Provider}NetworkException` | 결과 불확정 또는 서비스 불가 (`UNKNOWN` / `UNAVAILABLE`) | `NETWORK_ERROR` |
| `{Provider}AuthException` | 결과 불확정 (`UNKNOWN`) | `AUTH_ERROR` (명시적으로 잡는 경우) |
| `{Provider}ResponseParsingException` | 결과 불확정 (`UNKNOWN`) | `PARSING_ERROR` |
| `{Provider}Exception` (sealed 부모) | 결과 불확정 (`UNKNOWN`) | `EXTERNAL_ERROR` |

```kotlin
// ✅ 계층 순서대로 catch (GiftCardPurchaseBurnAdapter 실제 구현)
override fun burn(request: BurnRequest): BurnResult =
    try {
        logInfo { "[GiftCard - 소각] 요청 - sourceRequestId=${request.sourceRequestId}" }
        val response = apiClient.burn(
            GiftCardBurnRequest(sourceServer = SOURCE_SERVER, sourceRequestId = request.sourceRequestId, pinNo = request.pinNo)
        )
        val data = response.data
        BurnResult(status = BurnStatus.SUCCEEDED, code = data?.approvalCode ?: "", ...)
    } catch (e: GiftCardApiException) {
        logWarn { "[GiftCard - 소각] API 오류 - sourceRequestId=${request.sourceRequestId}, code=${e.code}" }
        BurnResult(status = BurnStatus.FAILED, code = e.code, message = e.message)
    } catch (e: GiftCardServerException) {
        logWarn { "[GiftCard - 소각] 서버 오류 - httpStatus=${e.httpStatus}" }
        BurnResult(status = BurnStatus.UNKNOWN, code = "HTTP_${e.httpStatus}", message = e.message)
    } catch (e: GiftCardNetworkException) {
        logWarn { "[GiftCard - 소각] 네트워크 오류" }
        BurnResult(status = BurnStatus.UNKNOWN, code = "NETWORK_ERROR", message = e.message)
    } catch (e: GiftCardException) {
        logWarn { "[GiftCard - 소각] 외부 오류" }
        BurnResult(status = BurnStatus.UNKNOWN, code = "EXTERNAL_ERROR", message = e.message)
    }
```

```kotlin
// ❌ 예외를 그대로 전파하거나 Exception 한 방으로 잡는다
override fun validate(request: ValidateRequest): ValidateResult {
    val response = apiClient.validate(...) // 예외 전파 → 호출 측이 외부 예외 타입에 결합됨
    return ValidateResult(...)
}
```

---

## 외부 ErrorCode 번역 패턴

외부 API 에러코드 enum 구조·`fromCode()` 패턴·`toPortErrorCode()` 구현 규칙은 [errorcode-convention.md](errorcode-convention.md) 참고.

---

## 로깅

**규칙: 요청 시작·응답 수신·예외 발생 세 지점에 비즈니스 식별자 중심 로그를 남기고, 민감정보는 마스킹한다.**

```kotlin
// ✅ 단계별 로그 + 민감정보 마스킹
logInfo { "[GiftCard - 핀번호 유효성 검증] 요청 - pinNo=${request.pinNo.take(4)}****" }
val response = apiClient.validate(...)
logInfo { "[GiftCard - 핀번호 유효성 검증] 응답 - pinNo=${response.pinNo.take(4)}****, amount=${response.amount}" }

// 계좌번호 마스킹: 뒷 4자리만 노출
"accountNumber=${request.accountNumber.takeLast(4).padStart(request.accountNumber.length, '*')}"
```

- 로그 태그는 `[Provider - 기능]` 형식의 대괄호 prefix를 사용한다 (`[GiftCard - 핀번호 유효성 검증]`, `[BISCUIT LINK - 계좌검증]`).
- 예외 catch 블록은 `logWarn`으로 기록한다. 외부 서비스 오류 확실 시 `logError`를 사용한다.

---

## DTO 매핑

**규칙: Port 타입 ↔ 외부 DTO 매핑은 Adapter 내부에서 수행하고, Port 타입은 외부 스키마 세부를 노출하지 않는다.**

```kotlin
// ✅ Adapter가 매핑 책임을 갖는다 (GiftCardPinValidateAdapter 실제 구현)
val response = apiClient.validate(GiftCardValidateRequest(pinNo = request.pinNo))
ValidateResult(
    status = ValidateStatus.VALID,
    pinCode = response.pinNo,
    amount = BigDecimal.valueOf(response.amount),
    validFrom = LocalDate.parse(response.validFrom),
    validTo = LocalDate.parse(response.validTo),
    code = "OK",
    message = "핀 유효성 검증 성공",
)
```

복잡한 매핑은 `private fun toXxx()` / 확장 함수로 분리하되, **Adapter 파일 내에 둔다**.

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
- 로그에 핀번호·카드번호·계좌번호 전체를 출력하지 않는다.
- Port 타입에 외부 DTO(`GiftCardValidateResponse` 등)를 직접 노출하지 않는다.

---

## 체크리스트

- [ ] Adapter 이름이 `{Provider}{Function}Adapter` 패턴을 따르는가?
- [ ] 하나의 Outbound Port만 구현하는가?
- [ ] 생성자 의존성이 ApiClient(혹은 동일 Provider의 보조 컴포넌트)로 제한되는가?
- [ ] Provider 예외 계층을 순서대로 catch하고 각각 다른 status/code로 매핑했는가?
- [ ] 외부 API 에러코드 번역에 `{Provider}ErrorCode` enum을 사용했는가?
- [ ] 요청/응답 로그에 비즈니스 식별자가 포함되고, 민감정보가 마스킹됐는가?
- [ ] 외부 DTO가 Port 시그니처로 새어나가지 않는가?
- [ ] Mock이 필요한 경우 `@Profile("!local")` / `@Profile("local")`로 대칭 구성됐는가?
