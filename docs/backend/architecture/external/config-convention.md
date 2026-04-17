# Config / Properties 컨벤션

---

## 핵심 규칙

**Provider 단위로 `RestClient` 빈을 `@Qualifier`로 명명해 분리 구성하고, 연결 설정은 `@ConfigurationProperties`로 바인딩한다.**

`RestClient`를 Provider별로 분리해야 타임아웃·인터셉터·baseUrl이 서로 간섭하지 않는다. 설정값은 코드 리터럴 대신 외부 구성 파일(`application.yml`)로 주입해 환경별 베이스 URL·타임아웃을 손쉽게 전환할 수 있어야 한다.

---

## 네이밍 규칙

| 항목 | 패턴 | 예시 |
|------|------|------|
| Config 클래스 | `{Provider}ServiceConfig` | `GiftCardServiceConfig` |
| Properties 클래스 | `{Provider}ServiceProperties` | `GiftCardServiceProperties` |
| `@ConfigurationProperties` prefix | `{provider}.service` (lower-dot) | `giftcard.service` |
| RestClient 빈 이름 | `{provider}ServiceRestClient` (camelCase) | `giftCardServiceRestClient` |
| `@Qualifier` 값 | 빈 이름과 동일 | `@Qualifier("giftCardServiceRestClient")` |

---

## Properties 클래스

**규칙: `data class`로 선언하고 `baseUrl`·`connectTimeout`·`readTimeout`을 기본 3종으로 둔다.**

`@ConfigurationProperties` 바인딩에 Kotlin `data class`를 사용하면 불변성과 기본값이 자연스럽게 표현된다. 타임아웃은 `Duration`으로 받아 `application.yml`에서 `PT5S` / `30s` 표기를 지원한다.

```kotlin
// ✅ data class + Duration + 합리적 기본값
@ConfigurationProperties(prefix = "giftcard.service")
data class GiftCardServiceProperties(
    val baseUrl: String = "http://localhost:8082",
    val connectTimeout: Duration = Duration.ofSeconds(5),
    val readTimeout: Duration = Duration.ofSeconds(30),
)
```

```kotlin
// ❌ 일반 class + Long(ms) → application.yml 가독성 저하, 단위 실수 유발
@ConfigurationProperties("giftcard.service")
class GiftCardServiceProperties {
    var baseUrl: String = ""
    var connectTimeoutMs: Long = 5000
    var readTimeoutMs: Long = 30000
}
```

- Provider가 인증 키(apiKey, secretKey)를 요구하면 Properties에 필드로 추가하고, 기본값은 비워둔다(운영에서 환경 변수로 주입).
- 기본값은 **로컬 개발 환경**을 기준으로 적는다(예: `http://localhost:8082`).

---

## Config 클래스

**규칙: `@Configuration` + `@EnableConfigurationProperties`로 Properties를 활성화하고, `@Bean`으로 `RestClient`를 생성한다.**

Properties를 자동 주입하려면 Config에서 `@EnableConfigurationProperties`를 선언하거나, `@ConfigurationPropertiesScan`이 적용된 상위 모듈에서 스캔되어야 한다. Provider 단위 Config에서 명시적으로 활성화하는 편이 의존 관계가 명확하다.

```kotlin
// ✅ Properties 활성화 + RestClient 빈 생성
@Configuration
@EnableConfigurationProperties(GiftCardServiceProperties::class)
class GiftCardServiceConfig {

    @Bean("giftCardServiceRestClient")
    fun giftCardServiceRestClient(properties: GiftCardServiceProperties): RestClient =
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
```

```kotlin
// ❌ Properties 활성화 누락 → 바인딩 실패
@Configuration
class GiftCardServiceConfig {
    @Bean
    fun giftCardServiceRestClient(): RestClient =
        RestClient.builder().baseUrl("http://localhost:8082").build() // 하드코딩
}
```

### 타임아웃 정책

- `connectTimeout`: 5초 전후(로컬/사내 서비스 기준). 외부 VPN/프록시 환경에서 긴 connect latency가 예상되면 늘린다.
- `readTimeout`: 30초 전후. 배치·결제 승인 등 응답이 긴 API는 Provider 단위로 별도 조정한다.
- 타임아웃은 반드시 설정한다. 기본값(무한 대기)은 장애 시 스레드 풀을 고갈시킨다.

---

## `@Qualifier` 주입

**규칙: ApiClient는 `@Qualifier`로 Provider 전용 빈을 지목해 주입받는다.**

빈 이름이 충돌하지 않도록 `{provider}ServiceRestClient` 네이밍을 지키고, ApiClient 생성자에서 같은 이름을 `@Qualifier`로 참조한다.

```kotlin
// ✅ 명시적 Qualifier
@Component
class GiftCardServiceApiClient(
    @Qualifier("giftCardServiceRestClient") private val restClient: RestClient,
    private val objectMapper: ObjectMapper,
)
```

```kotlin
// ❌ 타입 기반 주입 → 다른 Provider의 RestClient 빈과 충돌 시 모호함
@Component
class GiftCardServiceApiClient(
    private val restClient: RestClient, // 어떤 Provider의 빈인지 불명확
)
```

---

## ObjectMapper 사용

**규칙: `ObjectMapper`는 Spring Boot 기본 빈을 주입받아 재사용한다. Provider 전용 Mapper가 필요하면 별도 빈으로 분리한다.**

공통 Mapper는 Jackson Kotlin 모듈·JavaTime 등록 등이 이미 구성돼 있다. Provider가 특수 직렬화 규칙을 요구하는 경우에만 전용 `@Bean("giftCardObjectMapper")`를 만들고 `@Qualifier`로 주입한다.

---

## AutoConfiguration 등록

**규칙: external 모듈은 `ExternalAutoConfiguration` + `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`를 통해 Config가 자동 등록되도록 유지한다.**

신규 Provider Config가 같은 패키지 계층(`com.wooyong.musinsa.*`) 안에 있다면 컴포넌트 스캔만으로 주워간다. 외부 모듈에서 배제된 경로에 Config를 두면 수동 등록이 필요해지므로 패키지 규칙을 준수한다.

---

## 금지 사항

- `RestClient`를 Config 없이 ApiClient 내부에서 직접 `builder().build()`로 만들지 않는다.
- Properties 없이 `@Value("${giftcard.service.base-url}")`로 개별 값을 주입받지 않는다.
- 타임아웃 설정을 생략하지 않는다.
- 두 Provider가 같은 `RestClient` 빈을 공유하지 않는다. 빈 이름은 Provider별로 유일하게 둔다.
- 하드코딩된 baseUrl을 Config에 남기지 않는다. Properties 기본값을 사용한다.
- 민감 정보(apiKey, secretKey)를 Properties 기본값에 채우지 않는다.

---

## 체크리스트

- [ ] Properties가 `@ConfigurationProperties(prefix = "{provider}.service")` `data class`로 선언됐는가?
- [ ] `baseUrl`·`connectTimeout`·`readTimeout`이 기본 필드로 포함되고 `Duration` 타입인가?
- [ ] Config가 `@Configuration` + `@EnableConfigurationProperties`를 명시했는가?
- [ ] `RestClient` 빈 이름이 `{provider}ServiceRestClient` 패턴이고 ApiClient가 `@Qualifier`로 주입받는가?
- [ ] `connectTimeout` / `readTimeout`이 모두 설정됐는가?
- [ ] 민감정보 기본값이 Properties에 노출되지 않는가?
