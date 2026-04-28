# Logging Policy

백엔드 로깅 규칙.

---

## 로그 레벨

- **4xx** 응답: `WARN`
- **5xx** 응답: `ERROR` + stacktrace 포함
- 정상 요청: `INFO` 이하에서 필요한 범위로만

---

## MDC 키

요청 스코프로 전파되는 MDC 키:

- `tenantId` — 테넌트 식별자
- `requestId` — 요청 추적 ID

---

## 로그 포맷

Grafana + Loki 환경에서는 **JSON 대신 logfmt 스타일(`key=value`)** 평문을 사용한다.

- Loki는 로그 본문을 인덱싱하지 않고 라벨만 인덱싱하므로, 포맷이 JSON일 필요가 없다.
- 사람이 터미널에서 `tail -f`로 바로 읽을 수 있어 **로컬 디버깅이 빠르다**.
- `logfmt` 형태는 LogQL의 `| logfmt` 파서로 필드 추출이 가능해 운영 모니터링도 문제없다.

### Logback 패턴 (`backend/app/*/src/main/resources/logback-spring.xml`)

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [tenantId=%X{tenantId:-?} requestId=%X{requestId:-?}] %logger{40} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

### 출력 예시

```
2026-04-17 14:23:45.123 INFO  [tenantId=shop-a requestId=abc-123] c.e.a.CreateOrderUseCase - [ORDER] 주문 검증 시작 - tenantId=shop-a, productId=15, requestedQty=2
2026-04-17 14:23:45.456 WARN  [tenantId=shop-a requestId=def-456] c.e.a.CreateOrderUseCase - [ORDER] 재고 부족 - tenantId=shop-a, productId=15, requested=2, available=1
2026-04-17 14:23:45.789 ERROR [tenantId=shop-b requestId=ghi-789] c.e.PaymentController - [PAYMENT] 결제 실패 - orderId=88, amount=50000
    com.example.CoreException: ...
```

---

## Kotlin 사용 예시

프로젝트 전역에서 `LogExtension.kt`의 확장 함수 `logInfo` / `logWarn` / `logError` / `logDebug`를 사용한다. raw `LoggerFactory.getLogger(...)`는 쓰지 않는다.

### 왜 확장 함수인가

- **지연 평가**: 람다를 받으므로 해당 레벨이 비활성화된 경우 메시지 문자열이 아예 생성되지 않는다.
- **자동 logger 이름**: `inline + reified T`로 호출 클래스가 컴파일 시점에 결정되어 정확한 logger 이름이 찍힌다.
- **선언 생략**: 클래스마다 `private val log = LoggerFactory.getLogger(...)` 선언이 필요 없다.
- **자유로운 문자열 보간**: 람다 내부이므로 Kotlin `${...}` 보간을 써도 성능 손실이 없다.

### 기본 사용

```kotlin
import com.example.application.common.log.logInfo
import com.example.application.common.log.logWarn
import com.example.application.common.log.logError

class CreateGiftCardPurchaseUseCase(
    private val purchaseRepository: GiftCardPurchaseRepository,
) {
    fun execute(request: PurchaseRequest) {
        logInfo {
            "[GIFT-CARD-PURCHASE] 매입 신청 검증 시작 - tenantId=${request.tenantId}, " +
                "bankCode=${request.bankCode}, giftCardTypeId=${request.giftCardTypeId}"
        }

        if (request.amount > limit) {
            logWarn {
                "[GIFT-CARD-PURCHASE] 매입 한도 초과 - tenantId=${request.tenantId}, " +
                    "requestedAmount=${request.amount}, limit=$limit"
            }
        }
    }
}
```

### 예외 로깅

```kotlin
try {
    paymentGateway.charge(order)
} catch (e: PaymentException) {
    logError(cause = e) {
        "[PAYMENT] 결제 실패 - orderId=${order.id}, amount=${order.amount}"
    }
    throw e
}
```

`cause`를 전달하면 stacktrace가 출력에 자동 포함된다.

### 메시지 구조

모든 로그 메시지는 다음 구조를 따른다.

```
[SCOPE] 한국어 행위 설명 - key1=value1, key2=value2, ...
```

| 부분 | 규칙 |
|------|------|
| `[SCOPE]` | 도메인·기능 범위 태그. 대문자 + 하이픈. 예: `[GIFT-CARD-PURCHASE]`, `[PAYMENT]`, `[TENANT-ONBOARDING]` |
| 행위 설명 | 무엇을 하고 있거나 무엇이 일어났는지 한국어로 간결히 |
| `key=value, ...` | ` - ` 뒤에 `, `로 연결된 파라미터. LogQL `\| logfmt` 파서로 필드 추출 가능 |

### 키 네이밍 규칙

- camelCase 사용 (`orderId`, `customerId`, `productId`)
- 값에 공백이 포함되면 `key="value with space"` 형식으로 따옴표
- MDC 키(`tenantId`, `requestId`)는 Logback 패턴에서 자동 출력되지만, 파라미터 맥락 강조를 위해 메시지에 다시 명시해도 무방하다 (예제 참조)

### 안티 패턴

```kotlin
// ❌ 확장 함수 없이 raw logger 사용 → 선언 중복 + 지연 평가 상실
private val log = LoggerFactory.getLogger(MyClass::class.java)
log.info("order created: $orderId")

// ❌ [SCOPE] 태그 누락 → 도메인별 grep/LogQL 추적이 어렵다
logInfo { "매입 신청 검증 시작 - tenantId=$tenantId" }

// ❌ 파라미터를 자연어에 섞기 → logfmt 파싱 불가
logInfo { "[GIFT-CARD-PURCHASE] 테넌트 ${tenantId}에서 ${bankCode} 은행으로 신청" }

// ❌ ` - ` 구분자 누락 → 설명과 파라미터 경계가 흐려진다
logInfo { "[GIFT-CARD-PURCHASE] 매입 신청 검증 시작 tenantId=$tenantId" }
```

---

## 민감 데이터 차단

- 개인정보, 비밀번호, 토큰, 카드 정보 등 민감 데이터는 **로그에 남기지 않는다**.
- DTO 로깅 시 민감 필드는 마스킹하거나 제외한다.
