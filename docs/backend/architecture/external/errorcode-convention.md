# ErrorCode 컨벤션

---

## 핵심 규칙

**외부 API 비즈니스 에러코드는 Provider 전용 enum으로 정의하고, `toPortErrorCode()`로 Port ErrorCode로 번역한다.**

외부 에러코드 문자열을 Adapter에서 직접 비교하면 외부 API 변경에 취약해지고 번역 로직이 분산된다. `{Provider}ErrorCode` enum이 외부 코드와 Port ErrorCode 간의 단일 번역 테이블 역할을 담당한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 기본 ErrorCode | `{Provider}ErrorCode` | `GiftCardErrorCode`, `BiscuitLinkErrorCode` |
| 기능별 ErrorCode | `{Provider}{Function}ErrorCode` | `GiftCardAccountVerifyErrorCode` |
| 조회 메서드 | `fromCode(code: String)` (companion object) | 전 Provider 동일 |
| 번역 메서드 | `toPortErrorCode()` (private extension function) | Adapter 파일 내 정의 |

---

## 파일 위치

**규칙: ErrorCode enum은 DTO 파일과 분리해 `{Provider}ErrorCode.kt`로 독립 관리한다.**

에러코드를 DTO 파일에 포함하면 순수 데이터 홀더 역할이 흐려진다. Provider 패키지에 별도 파일로 둔다. 기능별로 에러코드 집합이 완전히 다른 경우 `{Provider}{Function}ErrorCode`로 분리한다.

---

## enum 구조

**규칙: enum 항목은 `code` 필드를 갖고, `fromCode()`로 외부 코드 문자열을 enum으로 변환한다.**

```kotlin
// ✅ 기본 구조
enum class GiftCardErrorCode(val code: String) {
    GIFT_CARD_NOT_FOUND("GIFT_CARD_NOT_FOUND"),
    GIFT_CARD_PIN_NOT_USABLE("GIFT_CARD_PIN_NOT_USABLE"),
    ;
    companion object {
        fun fromCode(code: String): GiftCardErrorCode? = entries.find { it.code == code }
    }
}

// ✅ 에러코드 집합이 다른 API는 기능별로 분리
enum class GiftCardAccountVerifyErrorCode(val code: String) {
    HOLDER_MISMATCH("HOLDER_MISMATCH"),
    ACCOUNT_NOT_FOUND("ACCOUNT_NOT_FOUND"),
    ACCOUNT_SUSPENDED("ACCOUNT_SUSPENDED"),
    SERVICE_UNAVAILABLE("SERVICE_UNAVAILABLE"),
    ;
    companion object {
        fun fromCode(code: String): GiftCardAccountVerifyErrorCode? = entries.find { it.code == code }
    }
}
```

- `fromCode()`는 매핑되지 않는 코드를 `null`로 반환한다. 호출부(Adapter)에서 null을 폴백 ErrorCode로 처리한다.

---

## Port ErrorCode 번역

**규칙: Adapter 내 `toPortErrorCode()` private extension function에서 enum 항목을 Port ErrorCode로 매핑하고, `null`(미매핑)은 폴백 코드로 처리한다.**

```kotlin
// ✅ enum 기반 번역
} catch (e: GiftCardApiException) {
    val externalErrorCode = GiftCardErrorCode.fromCode(e.code)
    ValidateResult(
        status = ValidateStatus.INVALID,
        errorCode = externalErrorCode.toPortErrorCode(),
        code = e.code,
        message = e.message,
    )
}

private fun GiftCardErrorCode?.toPortErrorCode(): ErrorCode =
    when (this) {
        GiftCardErrorCode.GIFT_CARD_NOT_FOUND      -> ErrorCode.NOT_FOUND
        GiftCardErrorCode.GIFT_CARD_PIN_NOT_USABLE -> ErrorCode.NOT_USABLE
        null                                       -> ErrorCode.UNAVAILABLE
    }
```

```kotlin
// ❌ 에러코드 문자열 직접 비교 → 외부 코드 변경에 취약, 번역 로직 분산
if (e.code == "GIFT_CARD_NOT_FOUND") { ErrorCode.NOT_FOUND }
else if (e.code == "GIFT_CARD_PIN_NOT_USABLE") { ErrorCode.NOT_USABLE }
```

- `when` 분기에 모든 enum 항목을 명시해 미매핑 케이스를 컴파일 시점에 강제한다. `else` 사용을 금지한다.
- `toPortErrorCode()`는 Adapter 파일 내 private extension function으로 둔다. ErrorCode enum 파일에 Port 의존성이 생기지 않도록 한다.

---

## 금지 사항

- 외부 에러코드 문자열을 Adapter에서 직접 비교하지 않는다. enum의 `fromCode()`를 경유한다.
- `fromCode()`에서 예외를 던지지 않는다. `null`을 반환하고 호출부가 폴백을 처리한다.
- 에러코드 enum을 DTO 파일 안에 포함하지 않는다. 별도 파일로 분리한다.
- `toPortErrorCode()`에서 `else` 분기를 사용하지 않는다. 신규 에러코드 추가 시 컴파일 오류로 누락을 감지한다.
- `toPortErrorCode()`를 ErrorCode enum 파일에 정의하지 않는다. Port 계층 의존성이 external enum에 침투한다.

---

## 체크리스트

- [ ] ErrorCode enum이 `{Provider}ErrorCode.kt` 별도 파일로 분리됐는가?
- [ ] enum 항목이 `code` 필드와 `fromCode()` companion을 갖추고 있는가?
- [ ] Adapter에서 외부 에러코드 문자열을 직접 비교하지 않고 `fromCode()`를 경유하는가?
- [ ] `toPortErrorCode()`에서 `else` 없이 모든 enum 항목을 `when`으로 명시했는가?
- [ ] `null`(미매핑 코드)을 폴백 ErrorCode로 처리하는가?
- [ ] `toPortErrorCode()`가 Adapter 파일 내 private extension function으로 정의됐는가?
