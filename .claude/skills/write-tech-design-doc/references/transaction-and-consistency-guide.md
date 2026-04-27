# 트랜잭션 경계 및 정합성 참조 가이드

이 문서는 TDD 7단계(트랜잭션 경계 및 정합성 설계) 작성 시 참조하는 상세 가이드다.

---

## 목차
1. [트랜잭션 경계 설정 원칙](#1-트랜잭션-경계-설정-원칙)
2. [트랜잭션 어노테이션 사용 패턴](#2-트랜잭션-어노테이션-사용-패턴)
3. [정합성 수준 선택 기준](#3-정합성-수준-선택-기준)
4. [이벤트 기반 최종 일관성 패턴](#4-이벤트-기반-최종-일관성-패턴)
5. [분산 락 적용 가이드](#5-분산-락-적용-가이드)
6. [TDD 트랜잭션 섹션 작성 예시](#6-tdd-트랜잭션-섹션-작성-예시)

---

## 1. 트랜잭션 경계 설정 원칙

### 기본 규칙
- **Service 클래스 단위**로 `@Transactional`을 적용한다
- Command Service: `@Transactional` (읽기/쓰기)
- Query Service: `@Transactional(readOnly = true)` (읽기 전용)
- **Facade**: 여러 Service를 묶는 상위 트랜잭션 경계

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
    // 모든 public 메서드에 @Transactional 적용
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
- 금전적 가치와 직결될 때 (결제, 재고, 포인트)
- **구현**: 단일 트랜잭션 + 분산 락

### 최종 일관성 (Eventual Consistency)
- 약간의 지연이 허용될 때
- 도메인 간 느슨한 결합이 중요할 때
- **구현**: 이벤트 기반 비동기 처리

### 판단 매트릭스

| 도메인 | 정합성 수준 | 근거 |
|--------|-----------|------|
| 재고 차감 | 강한 일관성 | 초과 판매 시 금전적 손실 |
| 포인트 차감 | 강한 일관성 | 이중 차감 시 클레임 |
| 주문 통계 | 최종 일관성 | 실시간성 불필요, 배치 집계 |
| 알림 발송 | 최종 일관성 | 지연 허용, 실패 시 재시도 |
| 검색 인덱스 | 최종 일관성 | 수초 지연 허용 |

---

## 4. 이벤트 기반 최종 일관성 패턴

### TransactionalEventListener 패턴

이 프로젝트에서 도메인 이벤트는 Spring의 `@TransactionalEventListener`를 사용한다.

```kotlin
// 이벤트 정의
data class OrderCancelledEvent(
    val orderId: Long,
    val tenantId: Long,
    val reason: String
)

// 이벤트 발행 (Service에서)
@Service
@Transactional
class OrderCancellationService(
    private val applicationEventPublisher: ApplicationEventPublisher
) {
    fun cancel(request: CancelRequest) {
        // ... 취소 처리
        applicationEventPublisher.publishEvent(
            OrderCancelledEvent(order.id, order.tenantId, request.reason)
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

### TDD에 이벤트 흐름 기술 시 포함할 항목

```markdown
### 이벤트 처리

| 이벤트 | 발행 조건 | 발행 시점 | 구독자 | 처리 내용 | 실패 시 대응 |
|--------|----------|----------|--------|----------|------------|
| OrderCancelledEvent | 주문 취소 완료 | AFTER_COMMIT | RefundHandler | 환불 처리 | 수동 환불 처리 + 알림 |
| | | | StockHandler | 재고 복원 | 재시도 3회 후 알림 |
```

---

## 5. 분산 락 적용 가이드

이 프로젝트는 Redisson 기반 `@DistributedLock`을 사용한다.

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
    key = "stock:#request.productVariantId:#tenantId",
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
- **세분화**: 가능한 한 좁은 범위의 키 사용 (productVariantId 단위, product 단위 X)
- **테넌트 격리**: 키에 tenantId 포함하여 테넌트 간 경합 방지
- **명명 규칙**: `{도메인}:{식별자}:{tenantId}` 형태

### TDD에 분산 락 기술 시 포함할 항목

```markdown
### 분산 락 설계

| 경합 자원 | 락 키 | waitTime | leaseTime | 사유 |
|----------|-------|----------|-----------|------|
| 상품 재고 | stock:{variantId}:{tenantId} | 3s | 10s | 재고 정합성 보장, 프로모션 시 동시 주문 대응 |
| 쿠폰 사용 | coupon:{couponId}:{tenantId} | 2s | 5s | 쿠폰 중복 사용 방지, 처리 시간 짧음 |
```

---

## 6. TDD 트랜잭션 섹션 작성 예시

```markdown
## 5. 트랜잭션 설계

### 5.1 트랜잭션 경계

| 연산 | 시작점 | 범위 | 격리 수준 | 사유 |
|------|--------|------|----------|------|
| 주문 생성 | OrderPaymentFacade | Order + OrderItem + Stock | READ_COMMITTED | 주문-재고가 원자적으로 처리되어야 초과 판매 방지 |
| 주문 조회 | QueryOrderService | Order (ReadOnly) | READ_COMMITTED | 읽기 전용, 스냅샷 읽기 |
| 주문 취소 | OrderCancellationService | Order + OrderCs | READ_COMMITTED | 취소와 CS 기록이 함께 저장되어야 이력 추적 가능 |
| 환불 처리 | (이벤트) RefundHandler | Refund | READ_COMMITTED | 취소 커밋 후 비동기, 실패 시 수동 처리 |

### 5.2 정합성 보장 전략

**강한 일관성 적용 영역**:
- 재고 차감: 분산 락 `@DistributedLock(key = "stock:{variantId}:{tenantId}")`
  → 프로모션 시 동시 주문 100건+ 대응
- 주문-결제 연결: 단일 Facade 트랜잭션
  → 결제 없는 주문 생성 방지

**최종 일관성 적용 영역**:
- 주문 취소 → 환불: `@TransactionalEventListener(AFTER_COMMIT)`
  → 취소는 즉시 반영, 환불은 PG사 응답 대기 필요하므로 비동기
- 주문 생성 → 통계 갱신: 이벤트 + 배치
  → 실시간성 불필요, 일 1회 집계

### 5.3 이벤트 처리

| 이벤트 | 발행 조건 | 발행 시점 | 구독자 | 처리 내용 | 실패 시 대응 |
|--------|----------|----------|--------|----------|------------|
| OrderCancelledEvent | 주문 취소 완료 | AFTER_COMMIT | RefundHandler | PG사 환불 요청 | 3회 재시도 후 수동 처리 대기열 |
| | | | StockRestoreHandler | 재고 복원 | 재시도 후 알림 |
| | | | NotificationHandler | 취소 알림 발송 | 실패 무시 (비핵심) |
```
