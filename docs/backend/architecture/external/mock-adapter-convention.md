# Mock Adapter 컨벤션

---

## 핵심 규칙

**실 Adapter와 동일 Port를 구현하는 Mock Adapter를 함께 제공하고, `@Profile("local")` / `@Profile("!local")`로 빈 선택을 분기한다.**

외부 시스템이 로컬에 존재하지 않거나 비용·보안상 호출이 불가능한 환경(로컬 개발)에서 흐름을 재현하려면 Port 구현이 필요하다. Mock Adapter는 실 구현과 동일한 인터페이스를 유지하되 고정된 시나리오를 반환해, 상위 계층이 Profile만 바꿔 실 호출과 Mock 응답을 자유롭게 교환할 수 있게 한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 클래스 | `Mock{Function}Adapter` | `MockBurnAdapter`, `MockPinValidateAdapter`, `MockPayoutAdapter`, `MockAccountVerificationAdapter` |
| 구현 Port | 실 Adapter와 동일 | `GiftCardPurchaseBurnPort`, `GiftCardPurchasePayoutPort` |
| 파일 배치 | 실 Adapter와 같은 Provider 패키지 | `im.bigs.ecommerce.giftcard/`, `im.bigs.ecommerce.biscuitlink/` |

> Provider prefix 없이 `Mock{Function}Adapter` 형태로 명명한다. `GiftCardPurchaseMockBurnAdapter` ❌ → `MockBurnAdapter` ✅

---

## Profile 분기

**규칙: 실 Adapter는 `@Profile("!local")`, Mock Adapter는 `@Profile("local")`로 선언해 동일 Port의 빈 충돌을 막는다.**

```kotlin
// ✅ 대칭 Profile (실제 구현)
@Component
@Profile("!local")
class GiftCardPurchaseBurnAdapter(
    private val apiClient: GiftCardApiClient,
) : GiftCardPurchaseBurnPort { ... }

@Component
@Profile("local")
class MockBurnAdapter : GiftCardPurchaseBurnPort { ... }
```

```kotlin
// ❌ 실 Adapter에 Profile 미지정 → local에서도 실제 API 호출 + 빈 충돌
@Component
class GiftCardPurchaseBurnAdapter(...) : GiftCardPurchaseBurnPort { ... }

@Component
@Profile("local")
class MockBurnAdapter : GiftCardPurchaseBurnPort { ... }
// → NoUniqueBeanDefinitionException
```

- Mock이 필요 없는 Adapter(로컬에서도 실 호출이 허용되는 경우)는 Profile 제약을 걸지 않는다.
- Profile 정책이 변경되어야 한다면 두 클래스를 **동시에** 수정해 대칭성을 유지한다.

---

## 시나리오 분기

**규칙: 입력 파라미터의 특정 문자열 패턴으로 테스트 시나리오를 구분한다.**

Mock은 단순히 성공만 리턴하면 실 세계의 실패 분기를 검증할 수 없다. 입력 문자열에 약속된 토큰을 섞으면 테스터가 동일 엔드포인트로 다양한 결과를 유발할 수 있다.

```kotlin
// ✅ 핀번호 기반 토큰 분기 (MockBurnAdapter 실제 구현)
@Component
@Profile("local")
class MockBurnAdapter : GiftCardPurchaseBurnPort {
    override fun burn(request: BurnRequest): BurnResult =
        if (request.pinNo.contains("BURN_FAIL", ignoreCase = true)) {
            BurnResult(status = BurnStatus.FAILED, code = "PIN_BURN_FAILED", message = "Mock 핀 소각 실패")
        } else if (request.pinNo.contains("BURN_UNKNOWN", ignoreCase = true)) {
            BurnResult(status = BurnStatus.UNKNOWN, code = "PIN_BURN_UNKNOWN", message = "Mock 핀 소각 결과 미확정")
        } else {
            BurnResult(status = BurnStatus.SUCCEEDED, code = "OK", message = "Mock 핀 소각 성공", giftCardAmount = BigDecimal("10000.00"))
        }
}
```

```kotlin
// ✅ 예금주명 기반 토큰 분기 (MockAccountVerificationAdapter 실제 구현)
@Component
@Profile("local")
class MockAccountVerificationAdapter : GiftCardPurchaseAccountVerificationPort {
    override fun verify(request: VerifyRequest): VerifyResult {
        val suffix = request.accountNumber.takeLast(4)
        logInfo { "[MOCK-계좌검증] 요청 - bankCode=${request.bankCode}, suffix=$suffix" }
        return when {
            request.accountHolder.contains("NOT_FOUND", ignoreCase = true) ->
                failure(GiftCardAccountVerifyErrorCode.ACCOUNT_NOT_FOUND, "Mock 계좌를 찾을 수 없습니다")
            request.accountHolder.contains("MISMATCH", ignoreCase = true) ->
                failure(GiftCardAccountVerifyErrorCode.HOLDER_MISMATCH, "Mock 예금주가 일치하지 않습니다")
            else ->
                VerifyResult(status = VerifyResult.Status.SUCCESS, code = "OK", message = "Mock 예금주 일치")
        }
    }
}
```

```kotlin
// ❌ 항상 성공만 반환 → 실패 경로 미검증
override fun burn(request: BurnRequest): BurnResult =
    BurnResult(status = BurnStatus.SUCCEEDED, code = "OK", message = "Mock 핀 소각 성공")
```

### 시나리오 토큰 권장 규칙

| Status 목표 | 토큰 예시 | 적용 파라미터 |
|------------|---------|-------------|
| 성공 (기본) | 토큰 없음 | - |
| 비즈니스 실패 | `*_FAIL`, `INVALID`, `NOT_FOUND`, `MISMATCH`, `SUSPENDED` | pinNo, accountHolder 등 |
| 결과 불확정 | `*_UNKNOWN`, `UNAVAILABLE` | pinNo 등 |

토큰은 **대소문자 무시**(`ignoreCase = true`)로 검사한다.

---

## Mock 고정값

**규칙: Mock 성공 응답은 현실 범위 내의 고정값을 반환하고, 시간은 `LocalDate.now()` 기준 상대값으로 둔다.**

```kotlin
// ✅ 상대 시간 + 현실 범위 (MockPinValidateAdapter 실제 구현)
GiftCardPinValidatePort.ValidateResult(
    status = GiftCardPinValidatePort.ValidateStatus.VALID,
    pinCode = request.pinNo,
    amount = BigDecimal("50000"),
    validFrom = LocalDate.now(),
    validTo = LocalDate.now().plusYears(1),
    code = "OK",
    message = "Mock 핀 유효성 검증 성공",
)
```

```kotlin
// ❌ 고정 과거 날짜 / 무작위 값
validFrom = LocalDate.of(2020, 1, 1),
validTo = LocalDate.of(2020, 12, 31),  // 만료된 날짜
```

---

## 외부 호출·의존성 금지

**규칙: Mock Adapter는 ApiClient·Repository·다른 Adapter를 주입받지 않는다.**

Mock은 외부 시스템·상태에 의존하지 않는 순수 구현이어야 한다. 단, Mock이 내부 ErrorCode enum(`GiftCardAccountVerifyErrorCode` 등)을 참조하는 것은 허용한다. 이는 외부 시스템 호출이 아닌 순수 로직이다.

```kotlin
// ✅ 무의존 (Port만 구현)
@Component
@Profile("local")
class MockBurnAdapter : GiftCardPurchaseBurnPort { ... }

// ✅ 내부 enum 참조는 허용
class MockAccountVerificationAdapter : GiftCardPurchaseAccountVerificationPort {
    private fun failure(externalErrorCode: GiftCardAccountVerifyErrorCode, message: String): VerifyResult = ...
}
```

```kotlin
// ❌ ApiClient 주입 → Mock이 외부 서비스를 타게 됨
@Component
@Profile("local")
class MockBurnAdapter(
    private val apiClient: GiftCardApiClient, // ❌
) : GiftCardPurchaseBurnPort { ... }
```

---

## 로깅

Mock Adapter는 로컬 디버깅 목적으로 `logInfo`를 사용할 수 있다. 이때 `[MOCK-기능]` prefix를 사용해 실 Adapter 로그와 구분한다.

```kotlin
// ✅ Mock 전용 로그 태그 (MockAccountVerificationAdapter 실제 구현)
logInfo { "[MOCK-계좌검증] 요청 - bankCode=${request.bankCode}, suffix=$suffix" }
logInfo { "[MOCK-계좌검증] 응답 - suffix=$suffix, status=${result.status}" }
```

---

## 판단 기준

| 상황 | 대응 |
|------|------|
| 외부 호출 비용이 크거나 로컬에서 불가 | Mock Adapter 필수 |
| 로컬에서 실 API 호출이 자유로움 | Mock 불필요, 실 Adapter만 유지 (Profile 제약 없음) |
| 통합 테스트 전용 Fake가 필요 | `src/test` 쪽에 별도 Fake를 두고, `src/main`의 Mock Adapter와 분리 |

---

## 금지 사항

- 실 Adapter와 Mock Adapter의 Profile을 비대칭으로 두지 않는다.
- Mock Adapter에 ApiClient·Repository·다른 Port를 주입하지 않는다.
- Mock이 항상 성공만 반환하도록 만들지 않는다. 실패·불확정 분기를 반드시 제공한다.
- 만료될 수 있는 고정 날짜나 무작위 값을 Mock 응답에 넣지 않는다.
- Mock 전용 로직(시나리오 토큰)을 실 Adapter에 포함하지 않는다.

---

## 체크리스트

- [ ] Mock Adapter 이름이 `Mock{Function}Adapter` 패턴인가?
- [ ] 실 Adapter `@Profile("!local")` + Mock Adapter `@Profile("local")`로 대칭인가?
- [ ] Mock Adapter가 ApiClient·Repository·다른 Port를 주입받지 않는가?
- [ ] 입력 문자열 토큰으로 성공·실패·불확정 시나리오 분기를 제공하는가?
- [ ] 성공 응답의 시간값이 `LocalDate.now()` 기준 상대값으로 구성되는가?
- [ ] 토큰 검사가 대소문자 무시(`ignoreCase = true`)로 수행되는가?
- [ ] `[MOCK-기능]` prefix로 실 Adapter 로그와 구분되는가?
