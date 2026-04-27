# 트랜잭션 경계 및 정합성 정책

백엔드 서비스의 트랜잭션 경계 설정 및 데이터 정합성 유지를 위한 공통 정책.

---

## 목차
1. [트랜잭션 경계 설정 원칙](#1-트랜잭션-경계-설정-원칙)
2. [트랜잭션 어노테이션 사용 패턴](#2-트랜잭션-어노테이션-사용-패턴)
3. [정합성 수준 선택 기준](#3-정합성-수준-선택-기준)
4. [이벤트 기반 최종 일관성 패턴](#4-이벤트-기반-최종-일관성-패턴)
5. [분산 락 적용 가이드](#5-분산-락-적용-가이드)

---

## 1. 트랜잭션 경계 설정 원칙

### 기본 규칙
- **Service 클래스 단위**로 트랜잭션 경계를 설정한다
- Command(쓰기) 서비스: 읽기/쓰기 트랜잭션
- Query(읽기) 서비스: 읽기 전용 트랜잭션 (성능 최적화)
- **Facade**: 여러 Service를 조합하는 상위 트랜잭션 경계

### 경계 설정 판단 기준

| 질문 | 판단 |
|------|------|
| 이 연산이 실패하면 전체가 롤백되어야 하는가? | Yes → 하나의 트랜잭션 |
| 일부 실패 시 나머지는 커밋해도 되는가? | Yes → 분리된 트랜잭션 |
| 이 연산이 외부 시스템을 호출하는가? | Yes → 외부 호출을 트랜잭션 밖으로 분리 고려 |
| 트랜잭션이 길어져 락 경합이 우려되는가? | Yes → 트랜잭션 범위 최소화 |

### 트랜잭션 범위 최소화 원칙

```
[나쁜 예: 트랜잭션 안에서 외부 호출]
@Transactional
fun processPayment(request: PaymentRequest) {
    val order = orderPort.findById(request.orderId)     // DB 조회
    val pgResult = pgPort.requestPayment(request)       // ← 외부 PG 호출 (느림)
    order.completePayment(pgResult)                     // 도메인 로직
    orderPort.save(order)                               // DB 저장
}
// 문제: PG 호출 동안 DB 커넥션과 트랜잭션을 점유

[좋은 예: 외부 호출을 트랜잭션 밖으로]
fun processPayment(request: PaymentRequest) {
    val pgResult = pgPort.requestPayment(request)       // 외부 호출 (트랜잭션 밖)
    completePaymentInTransaction(request, pgResult)     // 트랜잭션 내 처리
}

@Transactional
fun completePaymentInTransaction(request: PaymentRequest, pgResult: PgResult) {
    val order = orderPort.findById(request.orderId)
    order.completePayment(pgResult)
    orderPort.save(order)
}
```

---

## 2. 트랜잭션 어노테이션 사용 패턴

### Command Service

```kotlin
@Service
@Transactional  // 클래스 레벨 기본 적용
class OrderService(
    private val orderPort: OrderPort
) : OrderUseCase {
    override fun create(request: OrderRequest.Create): OrderResponse.Create {
        // write operation
    }
}
```

### Query Service

```kotlin
@Service
@Transactional(readOnly = true)  // 읽기 전용 최적화
class QueryOrderService(
    private val queryOrderPort: QueryOrderPort
) : QueryOrderUseCase {
    override fun findById(request: QueryOrderRequest.FindById): QueryOrderResponse {
        // read-only operation
    }
}
```

### Facade (크로스 도메인)

```kotlin
@Service
class OrderPaymentFacade(
    private val orderService: OrderService,
    private val paymentService: PaymentService,
    private val stockService: StockService
) {
    @Transactional  // 여러 서비스를 하나의 트랜잭션으로 묶음
    fun processOrderPayment(request: OrderPaymentRequest) {
        val order = orderService.create(request.toOrderRequest())
        stockService.decrease(request.toStockRequest())
        paymentService.process(request.toPaymentRequest())
    }
}
```

---

## 3. 정합성 수준 선택 기준

### 강한 일관성 (Strong Consistency)
- 모든 변경이 즉시 반영되어야 할 때
- 금전적 가치와 직결될 때
- **구현**: 단일 트랜잭션 + 분산 락

### 최종 일관성 (Eventual Consistency)
- 약간의 지연이 허용될 때
- 도메인 간 느슨한 결합이 중요할 때
- **구현**: 이벤트 기반 비동기 처리

### 판단 매트릭스

| 시나리오 | 정합성 수준 | 근거 |
|--------|-----------|------|
| 재고/잔액 차감 | 강한 일관성 | 초과 소비 시 금전적 손실 |
| 외부 결제 처리 | 강한 일관성 | 이중 처리 시 클레임 |
| 집계/통계 갱신 | 최종 일관성 | 실시간성 불필요, 배치 집계 가능 |
| 알림/메일 발송 | 최종 일관성 | 지연 허용, 실패 시 재시도 |
| 검색 인덱스 갱신 | 최종 일관성 | 수초 지연 허용 |

---

## 4. 이벤트 기반 최종 일관성 패턴

도메인 이벤트는 애플리케이션 이벤트 발행 메커니즘을 활용하여 도메인 간 결합을 최소화한다.

```kotlin
// 이벤트 정의
data class OrderCancelledEvent(
    val orderId: Long,
    val reason: String
)

// 이벤트 발행
@Service
@Transactional
class OrderCancellationService(
    private val applicationEventPublisher: ApplicationEventPublisher
) {
    fun cancel(request: CancelRequest) {
        // ... 취소 처리
        applicationEventPublisher.publishEvent(
            OrderCancelledEvent(order.id, request.reason)
        )
    }
}

// 이벤트 구독
@Component
class OrderCancellationEventHandler(
    private val refundService: RefundService,
    private val stockService: StockService
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handle(event: OrderCancelledEvent) {
        refundService.processRefund(event.orderId)
        stockService.restore(event.orderId)
    }
}
```

### 이벤트 설계 시 고려사항

| 고려사항 | 설명 |
|---------|------|
| **발행 시점** | `AFTER_COMMIT`: 메인 트랜잭션 성공 후 (대부분의 경우) |
| | `BEFORE_COMMIT`: 메인 트랜잭션 내에서 (함께 롤백 필요 시) |
| **실패 처리** | AFTER_COMMIT 핸들러 실패 시 메인 트랜잭션은 이미 커밋됨 → 보상 로직 필요 |
| **순서 보장** | 같은 이벤트의 여러 핸들러 간 실행 순서는 보장되지 않음 |
| **재시도** | 핸들러 실패 시 자동 재시도 없음 → 필요 시 직접 구현 |

---

## 5. 분산 락 적용 가이드

멀티 인스턴스 환경에서 동시 쓰기 경합을 제어하기 위해 분산 락을 사용한다.

### 적용 판단 기준

| 조건 | 분산 락 필요 |
|------|------------|
| 동일 자원에 대한 동시 쓰기 | Yes |
| 멀티 인스턴스 환경에서 경합 | Yes |
| 읽기 전용 조회 | No |
| 단일 인스턴스 + DB 락으로 충분 | No (DB 락 사용) |

### 사용 패턴

```kotlin
@DistributedLock(
    key = "stock:{productVariantId}",
    waitTime = 3,    // 락 획득 대기 최대 3초
    leaseTime = 10   // 락 자동 해제 10초 (데드락 방지)
)
fun decreaseStock(request: StockRequest.Decrease) {
    val variant = productVariantPort.findById(request.productVariantId)
    variant.decreaseStock(request.quantity)
    productVariantPort.save(variant)
}
```

### 락 키 설계 원칙
- **세분화**: 가능한 한 좁은 범위의 키 사용 (variantId 단위, productId 단위 X)
- **리소스 격리**: 키에 충분한 식별자를 포함하여 의도치 않은 경합 방지
- **명명 규칙**: `{도메인}:{리소스-식별자}` 형태

### waitTime / leaseTime 설정 가이드

| 연산 특성 | waitTime | leaseTime | 근거 |
|----------|----------|-----------|------|
| 빠른 연산 | 3초 | 10초 | 처리 1초 이내, 여유 확보 |
| 중간 연산 | 5초 | 30초 | 다단계 처리, 외부 호출 가능 |
| 느린 연산 | 10초 | 60초 | 외부 시스템 응답 대기 포함 |

락 획득 실패 시 클라이언트에 "잠시 후 재시도" 안내를 반환한다.
