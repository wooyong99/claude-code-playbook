# Mock Adapter 컨벤션

---

## 핵심 규칙

**실 Adapter와 동일 Port를 구현하는 Mock Adapter를 함께 제공하고, `@Profile("local")` / `@Profile("!local")`로 빈 선택을 분기한다.**

외부 시스템이 로컬에 존재하지 않거나 비용·보안상 호출이 불가능한 환경(로컬 개발)에서 흐름을 재현하려면 Port 구현이 필요하다. Mock Adapter는 실 구현과 동일한 인터페이스를 유지하되 고정된 시나리오를 반환해, 상위 계층이 Profile만 바꿔 실 호출과 Mock 응답을 자유롭게 교환할 수 있게 한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 클래스 | `{Provider}{Function}MockAdapter` | `GiftCardPurchaseMockPinAdapter`, `GiftCardPurchaseMockPayoutAdapter` |
| 구현 Port | 실 Adapter와 동일 | `GiftCardPurchasePinPort`, `GiftCardPurchasePayoutPort` |
| 파일 배치 | 실 Adapter와 같은 Provider 패키지 | `{provider}/` (예: `giftcard/`) |

---

## Profile 분기

**규칙: 실 Adapter는 `@Profile("!local")`, Mock Adapter는 `@Profile("local")`로 선언해 동일 Port의 빈 충돌을 막는다.**

Profile 미지정 상태로 두 빈이 공존하면 `NoUniqueBeanDefinitionException`이 발생한다. Profile을 명시하면 Spring이 환경에 맞는 하나만 선택한다.

```kotlin
// ✅ 대칭 Profile
@Component
@Profile("!local")
class GiftCardPurchasePinAdapter(
    private val apiClient: GiftCardServiceApiClient,
) : GiftCardPurchasePinPort { ... }

@Component
@Profile("local")
class GiftCardPurchaseMockPinAdapter : GiftCardPurchasePinPort { ... }
```

```kotlin
// ❌ 실 Adapter에 Profile 미지정 → local에서도 실제 API 호출
@Component
class GiftCardPurchasePinAdapter(...) : GiftCardPurchasePinPort { ... }

@Component
@Profile("local")
class GiftCardPurchaseMockPinAdapter : GiftCardPurchasePinPort { ... }
// → NoUniqueBeanDefinitionException
```

- Mock이 필요 없는 Adapter(로컬에서도 실 호출이 허용되는 경우)는 Profile 제약을 걸지 않는다.
- Profile 정책이 변경되어야 한다면 두 클래스를 **동시에** 수정해 대칭성을 유지한다.

---

## 시나리오 분기

**규칙: 입력 파라미터의 특정 문자열 패턴(`contains("INVALID")` / `contains("BURN_FAIL")`)으로 테스트 시나리오를 구분한다.**

Mock은 단순히 성공만 리턴하면 실 세계의 실패 분기를 검증할 수 없다. 입력 문자열에 약속된 토큰을 섞으면 테스터가 동일 엔드포인트로 다양한 결과를 유발할 수 있다.

```kotlin
// ✅ 토큰 기반 시나리오 분기
override fun burn(request: BurnRequest): BurnResult =
    if (request.pinNo.contains("BURN_FAIL", ignoreCase = true)) {
        BurnResult(BurnStatus.FAILED, "PIN_BURN_FAILED", "Mock 핀 소각 실패")
    } else if (request.pinNo.contains("BURN_UNKNOWN", ignoreCase = true)) {
        BurnResult(BurnStatus.UNKNOWN, "PIN_BURN_UNKNOWN", "Mock 핀 소각 결과 미확정")
    } else {
        BurnResult(BurnStatus.SUCCEEDED, "OK", "Mock 핀 소각 성공", BigDecimal("10000.00"))
    }
```

```kotlin
// ❌ 항상 성공만 반환 → 실패 경로 미검증
override fun burn(request: BurnRequest): BurnResult =
    BurnResult(BurnStatus.SUCCEEDED, "OK", "Mock 핀 소각 성공")
```

### 시나리오 토큰 권장 규칙

| Status 목표 | 토큰 예시 |
|------------|---------|
| 성공 (기본) | 토큰 없음 |
| 비즈니스 실패 | `*_FAIL` (예: `BURN_FAIL`, `PAYOUT_FAIL`) |
| 결과 불확정 | `*_UNKNOWN` (예: `BURN_UNKNOWN`) |
| 입력 무효 | `INVALID` |

Port의 Status enum을 기준으로 토큰을 정하면 실 Adapter와 동일한 상태 집합을 재현할 수 있다. 토큰은 **대소문자 무시**(`ignoreCase = true`)로 검사한다.

---

## Mock 고정값

**규칙: Mock 성공 응답은 현실 범위 내의 고정값을 반환하고, 시간은 `LocalDate.now()` 기준 상대값으로 둔다.**

하드코딩된 과거 날짜는 테스트 시점에 만료되어 혼란을 준다. 반대로 무작위 값을 반환하면 재현성이 깨진다.

```kotlin
// ✅ 상대 시간 + 현실 범위
ValidateResult(
    status = ValidateStatus.VALID,
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
amount = BigDecimal.valueOf(Random.nextLong()),  // 재현 불가
```

---

## 외부 호출·의존성 금지

**규칙: Mock Adapter는 ApiClient·Repository·다른 Adapter를 주입받지 않는다.**

Mock은 외부 시스템·상태에 의존하지 않는 순수 구현이어야 한다. 의존성이 추가되면 로컬 환경에서 또 다른 구동 조건이 필요해져 Mock의 목적이 퇴색된다.

```kotlin
// ✅ 무의존
@Component
@Profile("local")
class GiftCardPurchaseMockPinAdapter : GiftCardPurchasePinPort { ... }
```

```kotlin
// ❌ ApiClient 주입 → Mock이 외부 서비스를 타게 됨
@Component
@Profile("local")
class GiftCardPurchaseMockPinAdapter(
    private val apiClient: GiftCardServiceApiClient, // ❌
) : GiftCardPurchasePinPort { ... }
```

---

## 로깅

Mock Adapter는 별도 로그를 남기지 않아도 된다. 로컬 디버깅 목적으로 필요하면 `logInfo`로 가볍게 남기되, 실 Adapter의 `[PROVIDER-DOMAIN]` 태그와 구분되도록 `[MOCK-PROVIDER-DOMAIN]` prefix를 쓴다.

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

- [ ] Mock Adapter 이름이 `{Provider}{Function}MockAdapter` 패턴인가?
- [ ] 실 Adapter `@Profile("!local")` + Mock Adapter `@Profile("local")`로 대칭인가?
- [ ] Mock Adapter가 ApiClient·Repository·다른 Port를 주입받지 않는가?
- [ ] 입력 문자열 토큰으로 성공·실패·불확정 시나리오 분기를 제공하는가?
- [ ] 성공 응답의 시간값이 `LocalDate.now()` 기준 상대값으로 구성되는가?
- [ ] 토큰 검사가 대소문자 무시(`ignoreCase = true`)로 수행되는가?
