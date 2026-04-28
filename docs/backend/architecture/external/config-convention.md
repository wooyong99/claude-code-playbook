# Config / Properties 컨벤션

---

## 핵심 규칙

**Provider 단위로 HTTP 클라이언트 빈을 `@Qualifier`로 명명해 분리 구성하고, 연결 설정은 `@ConfigurationProperties`로 바인딩한다.**

HTTP 클라이언트를 Provider별로 분리해야 타임아웃·인터셉터·baseUrl이 서로 간섭하지 않는다. 설정값은 코드 리터럴 대신 외부 구성 파일(`application.yml`)로 주입해 환경별 베이스 URL·타임아웃을 손쉽게 전환할 수 있어야 한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| Config 클래스 | `{Provider}Config` | `GiftCardConfig`, `BiscuitLinkConfig` |
| Properties 클래스 | `{Provider}Properties` | `GiftCardProperties`, `BiscuitLinkProperties` |
| `@ConfigurationProperties` prefix | `{provider}.service` (lower-dot 또는 kebab-case) | `giftcard.service`, `biscuit-link.service` |
| HTTP 클라이언트 빈 이름 | `{provider}ServiceRestClient` 또는 `{provider}ServiceClient` | `giftCardServiceRestClient`, `biscuitLinkServiceClient` |
| `@Qualifier` 값 | 빈 이름과 동일 | `@Qualifier("giftCardServiceRestClient")` |

> `{Provider}ServiceConfig` / `{Provider}ServiceProperties` 형식의 `Service` 접미사는 사용하지 않는다.

---

## Properties 클래스

**규칙: `data class`로 선언하고 `baseUrl`·`apiKey`·`connectTimeout`·`readTimeout`을 기본 필드로 둔다.**

`@ConfigurationProperties` 바인딩에 Kotlin `data class`를 사용하면 불변성과 기본값이 자연스럽게 표현된다. 타임아웃은 `Duration`으로 받아 `application.yml`에서 `PT5S` / `30s` 표기를 지원한다.

```kotlin
// ✅ data class + Duration + 합리적 기본값 (GiftCardProperties 실제 구현)
@ConfigurationProperties(prefix = "giftcard.service")
data class GiftCardProperties(
    val baseUrl: String = "http://localhost:8082",
    val apiKey: String = "",
    val connectTimeout: Duration = Duration.ofSeconds(5),
    val readTimeout: Duration = Duration.ofSeconds(30),
)

// ✅ kebab-case prefix도 허용 (BiscuitLinkProperties 실제 구현)
@ConfigurationProperties(prefix = "biscuit-link.service")
data class BiscuitLinkProperties(
    val baseUrl: String = "http://localhost:8083",
    val apiKey: String = "",
    val connectTimeout: Duration = Duration.ofSeconds(5),
    val readTimeout: Duration = Duration.ofSeconds(30),
)
```

```kotlin
// ❌ 일반 class + Long(ms) → 단위 실수 유발, application.yml 가독성 저하
@ConfigurationProperties("giftcard.service")
class GiftCardProperties {
    var baseUrl: String = ""
    var connectTimeoutMs: Long = 5000
}
```

- Provider가 API 키를 요구하면 `apiKey` 필드를 추가하고 기본값은 빈 문자열로 둔다(운영에서 환경 변수로 주입).
- 기본값은 **로컬 개발 환경**을 기준으로 적는다(예: `http://localhost:8082`).
- **민감 정보(실제 apiKey 값)를 기본값에 채우지 않는다.**

---

## Config 클래스

**규칙: `@Configuration` + `@EnableConfigurationProperties`로 Properties를 활성화하고, `@Bean`으로 HTTP 클라이언트 빈을 생성한다.**

```kotlin
// ✅ Properties 활성화 + RestClient 빈 생성 (GiftCardConfig 실제 구현)
@Configuration
@EnableConfigurationProperties(GiftCardProperties::class)
class GiftCardConfig {

    @Bean("giftCardServiceRestClient")
    fun giftCardServiceRestClient(properties: GiftCardProperties): RestClient =
        RestClient.builder()
            .baseUrl(properties.baseUrl)
            .requestFactory(
                SimpleClientHttpRequestFactory().apply {
                    setConnectTimeout(properties.connectTimeout)
                    setReadTimeout(properties.readTimeout)
                }
            )
            .build()
}

// ✅ defaultHeader가 필요한 Provider (BiscuitLinkConfig 실제 구현)
@Configuration
@EnableConfigurationProperties(BiscuitLinkProperties::class)
class BiscuitLinkConfig {

    @Bean("biscuitLinkServiceClient")
    fun biscuitLinkRestClient(properties: BiscuitLinkProperties): RestClient =
        RestClient.builder()
            .baseUrl(properties.baseUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, "application/json")
            .requestFactory(
                SimpleClientHttpRequestFactory().apply {
                    setConnectTimeout(properties.connectTimeout)
                    setReadTimeout(properties.readTimeout)
                }
            )
            .build()
}
```

```kotlin
// ❌ Properties 활성화 누락, 하드코딩 URL
@Configuration
class GiftCardConfig {
    @Bean
    fun giftCardServiceRestClient(): RestClient =
        RestClient.builder().baseUrl("http://localhost:8082").build()
}
```

### 타임아웃 정책

- `connectTimeout`: 5초 전후(로컬/사내 서비스 기준).
- `readTimeout`: 30초 전후. 배치·결제 승인 등 응답이 긴 API는 Provider 단위로 별도 조정한다.
- 타임아웃은 반드시 설정한다. 기본값(무한 대기)은 장애 시 스레드 풀을 고갈시킨다.

---

## AutoConfiguration 등록

**규칙: external 모듈은 `ExternalAutoConfiguration` + `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`를 통해 Config가 자동 등록되도록 유지한다.**

신규 Provider Config가 `im.bigs.ecommerce.*` 패키지 안에 있다면 컴포넌트 스캔만으로 주워간다.

---

## 금지 사항

- HTTP 클라이언트를 Config 없이 ApiClient 내부에서 직접 생성하지 않는다.
- Properties 없이 `@Value("${...}")`로 개별 값을 주입받지 않는다.
- 타임아웃 설정을 생략하지 않는다.
- 두 Provider가 같은 HTTP 클라이언트 빈을 공유하지 않는다. 빈 이름은 Provider별로 유일하게 둔다.
- 하드코딩된 baseUrl을 Config에 남기지 않는다. Properties 기본값을 사용한다.
- 민감 정보(실제 apiKey 값)를 Properties 기본값에 채우지 않는다.

---

## 체크리스트

- [ ] Properties가 `@ConfigurationProperties(prefix = "...")` `data class`로 선언됐는가?
- [ ] `baseUrl`·`connectTimeout`·`readTimeout`이 기본 필드로 포함되고 `Duration` 타입인가?
- [ ] `apiKey` 등 인증 필드가 있을 경우 기본값이 빈 문자열인가?
- [ ] Config가 `@Configuration` + `@EnableConfigurationProperties`를 명시했는가?
- [ ] HTTP 클라이언트 빈 이름이 Provider 전용 패턴(`{provider}ServiceRestClient` 등)인가?
- [ ] `connectTimeout` / `readTimeout`이 모두 설정됐는가?
- [ ] 민감정보 기본값이 Properties에 노출되지 않는가?
