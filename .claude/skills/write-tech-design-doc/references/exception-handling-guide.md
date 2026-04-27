# 예외 및 실패 처리 참조 가이드

이 문서는 TDD 8단계(예외·실패 처리 전략) 작성 시 참조하는 상세 가이드다.

---

## 목차
1. [예외 계층 구조](#1-예외-계층-구조)
2. [예외 사용 기준](#2-예외-사용-기준)
3. [실패 시나리오 도출 체크리스트](#3-실패-시나리오-도출-체크리스트)
4. [복구 전략 패턴](#4-복구-전략-패턴)
5. [멱등성 보장 패턴](#5-멱등성-보장-패턴)
6. [TDD 예외 섹션 작성 예시](#6-tdd-예외-섹션-작성-예시)

---

## 1. 예외 계층 구조

이 프로젝트는 다음 예외 계층을 사용한다.

```
RuntimeException
├── EIllegalArgumentException     // 비즈니스 전제조건 위반
│     - requireE() 함수로 발생
│     - HTTP 200 (OK) + 에러 메시지
│     - 클라이언트 재시도로 해결 가능한 오류
│
├── EIllegalStateException        // 비즈니스 상태 불일치
│     - checkE() 함수로 발생
│     - HTTP 200 (OK) + 에러 메시지
│     - 선행 조건 미충족 (상태 변경 필요)
│
├── EEntityNotFoundException      // 엔티티 미존재
│     - entityNotNull() 함수로 발생
│     - HTTP 500 (Internal Server Error)
│     - 존재해야 하는 데이터가 없음 (시스템 오류)
│
└── Status4xxException            // HTTP 상태 기반 예외
      - 인증/인가 실패 등
      - HTTP 4xx 응답
```

### 주의사항
- `EIllegalArgumentException`은 HTTP **200**을 반환한다. 일반적인 REST 관례(400)와 다르다.
  이 프로젝트에서는 "비즈니스 검증 실패는 정상 응답이나 결과가 실패"라는 관점을 사용한다.
- `EEntityNotFoundException`은 HTTP **500**을 반환한다.
  "존재해야 할 데이터가 없는 것은 시스템 오류"라는 관점이다.
- 테스트 작성 시 `IllegalArgumentException`이 아닌 **`EIllegalArgumentException`**을 사용해야 한다.

---

## 2. 예외 사용 기준

### 검증 함수 사용 기준

| 함수 | 용도 | 예외 타입 | 사용 계층 |
|------|------|----------|----------|
| `requireE(condition) { message }` | 입력값/전제조건 검증 | EIllegalArgumentException | Domain, Request DTO |
| `checkE(condition) { message }` | 상태/불변식 검증 | EIllegalStateException | Domain |
| `entityNotNull(value) { message }` | 엔티티 존재 확인 | EEntityNotFoundException | Adapter |

### 각 계층별 예외 처리 책임

| 계층 | 예외 발생 | 예외 처리 | 설명 |
|------|----------|----------|------|
| Domain | `requireE`, `checkE` | 발생만 함 | 비즈니스 규칙 위반 감지 |
| Application (Service) | 비즈니스 흐름 예외 | 발생만 함 | 흐름 중 예외 상황 감지 |
| Infrastructure (Adapter) | `entityNotNull` | 발생만 함 | 데이터 존재 확인 |
| Controller | 발생하지 않음 | GlobalExceptionHandler | 통합 예외 처리 |

### i18n 메시지 패턴

예외 메시지는 i18n 키를 사용한다.

```kotlin
requireE(productName.isNotBlank()) { "product.name.required".toI18n() }
requireE(price >= BigDecimal.ZERO) { "product.price.min_zero".toI18n() }
```

---

## 3. 실패 시나리오 도출 체크리스트

TDD에서 실패 시나리오를 빠짐없이 도출하기 위한 체크리스트.
해당 항목에 체크하고, 각 시나리오의 대응 전략을 기술한다.

### 입력 검증 실패
- [ ] 필수 필드 누락
- [ ] 필드 값 범위 초과 (길이, 최소/최대값)
- [ ] 잘못된 형식 (이메일, 전화번호 등)
- [ ] 존재하지 않는 참조 ID

### 비즈니스 규칙 위반
- [ ] 허용되지 않는 상태 전이
- [ ] 수량/금액 부족 (재고, 포인트, 잔액)
- [ ] 유효 기간 만료 (쿠폰, 이벤트)
- [ ] 중복 요청 (이미 처리된 주문 취소, 이미 사용된 쿠폰)
- [ ] 권한 부족 (다른 테넌트 데이터 접근)

### 데이터 정합성
- [ ] 동시 수정으로 인한 데이터 충돌
- [ ] 참조 데이터 삭제 후 접근
- [ ] 트랜잭션 타임아웃
- [ ] 데드락

### 외부 시스템 연동
- [ ] 외부 API 타임아웃
- [ ] 외부 API 오류 응답
- [ ] 네트워크 단절
- [ ] 외부 시스템 유지보수 중
- [ ] 응답 형식 불일치

### 인프라
- [ ] DB 커넥션 풀 고갈
- [ ] Redis 연결 실패 (분산 락, 캐시)
- [ ] 디스크 용량 부족 (파일 업로드)
- [ ] 메모리 부족 (대량 데이터 처리)

---

## 4. 복구 전략 패턴

### 즉시 실패 (Fail Fast)
비즈니스 검증 실패 시 즉시 예외를 던지고 처리를 중단한다.

```kotlin
fun cancelOrder(orderId: Long) {
    val order = orderPort.findById(orderId)
    requireE(order.status.isCancellable()) { "order.cancel.invalid_status".toI18n() }
    // 검증 통과 후에만 후속 처리
}
```

**적용 시나리오**: 입력 검증, 상태 검증, 권한 검증

### 재시도 (Retry)
일시적 실패에 대해 재시도한다.

```kotlin
@Retryable(
    value = [OptimisticLockingFailureException::class],
    maxAttempts = 3,
    backoff = Backoff(delay = 100, multiplier = 2.0)
)
fun updateStock(request: StockRequest) { ... }
```

**적용 시나리오**: 낙관적 잠금 충돌, 외부 API 일시적 오류, 네트워크 지연

### 보상 트랜잭션 (Compensating Transaction)
이미 커밋된 트랜잭션을 되돌리는 역연산을 수행한다.

```
주문 생성 흐름:
1. 재고 차감 (커밋)
2. 결제 요청 (실패!)
3. → 보상: 재고 복원

보상 흐름:
1. StockRestoreEvent 발행
2. StockRestoreHandler: 재고 원복
3. 복원 실패 시: 알림 + 수동 처리 대기열
```

**적용 시나리오**: 다단계 처리에서 후반부 실패, 외부 시스템 연동 실패 후 롤백

### 수동 처리 대기열 (Manual Queue)
자동 복구가 불가능한 경우, 관리자가 수동으로 처리할 수 있도록 대기열에 넣는다.

**적용 시나리오**: PG사 환불 실패, 외부 시스템 장기 장애, 데이터 불일치

---

## 5. 멱등성 보장 패턴

동일한 요청을 여러 번 처리해도 결과가 동일하도록 보장하는 패턴.

### 적용 판단 기준

| 연산 | 멱등성 필요 | 근거 |
|------|-----------|------|
| 결제 요청 | 필수 | 네트워크 타임아웃 후 재시도 시 이중 결제 방지 |
| 재고 차감 | 필수 | 이중 차감 방지 |
| 주문 상태 변경 | 필수 | 중복 요청 시 상태 꼬임 방지 |
| 알림 발송 | 권장 | 중복 알림 UX 저하 |
| 조회 | 자동 (GET) | 부작용 없음 |

### 멱등키 패턴

```kotlin
// 요청에 고유 키를 포함하여 중복 처리 방지
fun processPayment(idempotencyKey: String, request: PaymentRequest) {
    // 이미 처리된 요청인지 확인
    val existing = paymentPort.findByIdempotencyKey(idempotencyKey)
    if (existing != null) return existing.toResponse()  // 기존 결과 반환

    // 신규 처리
    val result = executePayment(request)
    paymentPort.save(result.copy(idempotencyKey = idempotencyKey))
    return result.toResponse()
}
```

### 상태 기반 멱등성

```kotlin
// 상태 전이를 검증하여 이미 처리된 상태면 무시
fun cancelOrder(orderId: Long) {
    val order = orderPort.findById(orderId)
    if (order.status == OrderStatus.CANCELLED) {
        return  // 이미 취소됨 → 멱등하게 처리
    }
    requireE(order.status.isCancellable()) { "취소 불가능한 상태" }
    order.cancel()
    orderPort.save(order)
}
```

---

## 6. TDD 예외 섹션 작성 예시

```markdown
## 6. 예외 및 실패 처리

### 6.1 예외 분류

| 예외 유형 | 예외 클래스 | 발생 조건 | HTTP | 사용자 메시지 |
|----------|-----------|----------|------|-------------|
| 주문 미존재 | EEntityNotFoundException | orderId에 해당하는 주문 없음 | 500 | error.entity_not_found |
| 취소 불가 상태 | EIllegalArgumentException | status가 PENDING/PAID가 아님 | 200 | order.cancel.invalid_status |
| 이미 취소됨 | - (멱등 처리) | status가 CANCELLED | 200 | 정상 응답 (기존 취소 결과) |
| 재고 복원 실패 | RuntimeException | 상품이 삭제된 경우 | - | 이벤트 핸들러에서 처리 |

### 6.2 실패 시나리오 및 복구 전략

| # | 시나리오 | 발생 가능성 | 영향 범위 | 복구 전략 |
|---|---------|-----------|----------|----------|
| 1 | 주문 취소 중 DB 타임아웃 | 낮음 | 단일 주문 | 자동 롤백, 재시도 안내 |
| 2 | 취소 후 PG 환불 실패 | 중간 | 단일 주문 | 3회 재시도 → 수동 처리 대기열 |
| 3 | 취소 후 재고 복원 실패 | 낮음 | 재고 정합성 | 재시도 → 관리자 알림 |
| 4 | 동시 취소 요청 (같은 주문) | 중간 | 단일 주문 | 분산 락으로 직렬화 |
| 5 | 취소 중 상품 삭제 | 낮음 | 재고 복원 불가 | 삭제된 상품 재고 복원 스킵 |

### 6.3 멱등성 보장

| 연산 | 보장 방식 | 설명 |
|------|----------|------|
| 주문 취소 | 상태 기반 | 이미 CANCELLED면 기존 결과 반환 |
| 환불 요청 | 멱등키 | refundKey로 중복 환불 방지 |
```
