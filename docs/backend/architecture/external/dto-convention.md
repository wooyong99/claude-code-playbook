# DTO 컨벤션

---

## 핵심 규칙

**외부 API용 DTO는 Provider의 JSON 스키마를 1:1로 표현하는 `data class`로 선언하고, 알 수 없는 필드 허용(`@JsonIgnoreProperties(ignoreUnknown = true)`) + 필드명 명시(`@JsonProperty`)를 모든 타입에 적용한다.**

외부 시스템의 스키마는 예고 없이 확장될 수 있다. 허용 정책을 명시하지 않으면 신규 필드 추가만으로 역직렬화가 깨진다. 또한 Kotlin 프로퍼티명과 외부 JSON 키가 스네이크/카멜 혼용으로 다를 수 있으므로 `@JsonProperty`로 매핑을 고정한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| 파일 | `{Provider}Dtos.kt` | `GiftCardDtos.kt`, `BiscuitLinkDtos.kt` |
| 요청 DTO | `{Provider}{Function}Request` | `GiftCardBurnRequest`, `GiftCardValidateRequest`, `BiscuitLinkVerifyAccountRequest` |
| 응답 DTO | `{Provider}{Function}Response` | `GiftCardBurnResponse`, `BiscuitLinkVerifyAccountResponse` |
| 응답 data payload | `{Provider}{Function}Data` | `GiftCardBurnData` |
| 공통 래퍼 | `{Provider}ApiResponse<T>` | `GiftCardApiResponse<T>`, `BiscuitLinkApiResponse<T>` |
| 공통 결과 | `{Provider}ResponseResult` | `GiftCardResponseResult` |

---

## 필수 어노테이션

**규칙: 모든 DTO에 `@JsonIgnoreProperties(ignoreUnknown = true)`를 달고, 모든 프로퍼티에 `@JsonProperty`를 명시한다.**

```kotlin
// ✅ 누락 없는 어노테이션 (GiftCardDtos.kt 실제 구현)
@JsonIgnoreProperties(ignoreUnknown = true)
data class GiftCardBurnRequest(
    @get:JsonProperty("sourceServer")
    val sourceServer: String,
    @get:JsonProperty("sourceRequestId")
    val sourceRequestId: String,
    @get:JsonProperty("groupParam")
    val groupParam: String? = null,
    @get:JsonProperty("pinNo")
    val pinNo: String,
)
```

```kotlin
// ❌ 어노테이션 누락 → 외부 스키마 확장 시 역직렬화 실패
data class GiftCardBurnRequest(
    val sourceServer: String,
    val sourceRequestId: String,
    val pinNo: String,
)
```

- `@JsonProperty`는 `@get:JsonProperty` 형태로 getter에 부착한다.
- 외부 API가 camelCase를 사용하면 그대로 `@JsonProperty("sourceServer")`처럼 적는다.

---

## 공통 응답 래퍼 구조

**규칙: 외부 스키마의 중첩 구조는 그대로 반영하고, 플랫하게 합치지 않는다.**

Provider마다 공통 응답 래퍼 구조가 다를 수 있다. DTO도 그 구조를 그대로 반영한다.

```kotlin
// ✅ GiftCard: { result: {...}, payload: T? } 구조
@JsonIgnoreProperties(ignoreUnknown = true)
data class GiftCardApiResponse<T>(
    @get:JsonProperty("result") val result: GiftCardResponseResult,
    @get:JsonProperty("payload") val payload: T? = null,
)

@JsonIgnoreProperties(ignoreUnknown = true)
data class GiftCardResponseResult(
    @get:JsonProperty("code") val code: Int,
    @get:JsonProperty("message") val message: String?,
    @get:JsonProperty("detailMessage") val detailMessage: String? = null,
    @get:JsonProperty("messageCode") val messageCode: String? = null,
)

// ✅ BiscuitLink: { code, message, payload: T? } 구조 (더 단순한 래퍼)
@JsonIgnoreProperties(ignoreUnknown = true)
data class BiscuitLinkApiResponse<T>(
    @get:JsonProperty("code") val code: Int,
    @get:JsonProperty("message") val message: String?,
    @get:JsonProperty("payload") val payload: T? = null,
)
```

```kotlin
// ❌ 플랫하게 병합 → 원 스키마 추적 불가
data class GiftCardBurnResult(
    val resultCode: Int,
    val approvalCode: String?,
    val price: Long?,
)
```

---

## 스키마 매핑 원칙

**규칙: payload 내부에 data 계층이 있으면 별도 data DTO로 표현한다.**

```kotlin
// ✅ payload → { code, message, data: GiftCardBurnData? } 구조 유지
@JsonIgnoreProperties(ignoreUnknown = true)
data class GiftCardBurnResponse(
    @get:JsonProperty("code") val code: String?,
    @get:JsonProperty("message") val message: String?,
    @get:JsonProperty("data") val data: GiftCardBurnData?,
)

@JsonIgnoreProperties(ignoreUnknown = true)
data class GiftCardBurnData(
    @get:JsonProperty("approvalCode") val approvalCode: String?,
    @get:JsonProperty("approvalDate") val approvalDate: String?,
    @get:JsonProperty("orderNo") val orderNo: String?,
    @get:JsonProperty("voucherName") val voucherName: String?,
    @get:JsonProperty("price") val price: Long?,
)
```

---

## 타입 선택

**규칙: 원 스키마의 타입을 그대로 유지하고, 도메인 타입(`BigDecimal`, `LocalDate` 등)으로의 변환은 Adapter가 수행한다.**

```kotlin
// ✅ 원 스키마 타입 유지 (GiftCardDtos.kt)
data class GiftCardValidateResponse(
    @get:JsonProperty("pinNo") val pinNo: String,
    @get:JsonProperty("amount") val amount: Long,     // Long → Adapter에서 BigDecimal.valueOf()
    @get:JsonProperty("validFrom") val validFrom: String, // String → Adapter에서 LocalDate.parse()
    @get:JsonProperty("validTo") val validTo: String,
)
```

```kotlin
// ❌ DTO에서 도메인 타입으로 미리 변환 → 역직렬화 실패 시 진단 어려움
data class GiftCardValidateResponse(
    val amount: BigDecimal,
    val validFrom: LocalDate,
)
```

### nullable 정책

- 외부가 생략 가능하다고 명시하거나 실무 관찰로 누락 가능성이 확인된 필드만 nullable로 둔다.
- `payload` 같은 래퍼 최상위는 실패 응답에서 null이 올 수 있으므로 nullable이 기본이다.
- 필수로 기대하지만 null이 오면 `ResponseParsingException`으로 승격한다.

---

## 배치 형태

**규칙: Provider의 모든 DTO는 한 파일(`{Provider}Dtos.kt`)에 모은다.**

관련 DTO를 파일 단위로 그룹화하면 스키마 변경 시 diff가 한 곳에 모인다.

---

## 금지 사항

- `@JsonIgnoreProperties(ignoreUnknown = true)`를 생략하지 않는다.
- `@JsonProperty`를 일부 필드만 붙이지 않는다. **전부 붙이거나 전부 생략하지 않는다**(일관성).
- DTO에서 외부 스키마에 없는 필드를 임의로 추가하지 않는다.
- DTO 클래스에 비즈니스 메서드(`isValid()` 등)를 추가하지 않는다. 순수 데이터 홀더로 유지한다.
- 외부 응답 구조를 플랫하게 병합해 중첩 계층을 생략하지 않는다.
- 에러코드 enum을 DTO 파일 안에 포함하지 않는다. 별도 파일로 분리한다.

---

## 체크리스트

- [ ] 모든 DTO에 `@JsonIgnoreProperties(ignoreUnknown = true)`가 달려 있는가?
- [ ] 모든 프로퍼티에 `@get:JsonProperty("...")`가 명시됐는가?
- [ ] 공통 래퍼(`GiftCardApiResponse<T>`, `BiscuitLinkApiResponse<T>`)가 외부 스키마 구조를 그대로 재현했는가?
- [ ] payload 내부의 data 계층도 별도 DTO(`{Provider}{Function}Data`)로 표현됐는가?
- [ ] 원 스키마 타입(Long/String)을 유지하고 도메인 타입 변환은 Adapter에 맡겼는가?
- [ ] Provider의 DTO가 한 파일에 모여 있는가?
- [ ] DTO에 비즈니스 로직이 섞이지 않았는가?
