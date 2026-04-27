# 아키텍처 설계 참조 가이드

이 문서는 TDD 5단계(아키텍처 설계 및 계층 분리) 작성 시 참조하는 상세 가이드다.

---

## 목차
1. [헥사고날 아키텍처 계층별 책임](#1-헥사고날-아키텍처-계층별-책임)
2. [CQRS 분리 기준 판단](#2-cqrs-분리-기준-판단)
3. [Facade 패턴 적용 기준](#3-facade-패턴-적용-기준)
4. [설계 대안 분석 작성법](#4-설계-대안-분석-작성법)
5. [처리 흐름 작성법](#5-처리-흐름-작성법)

---

## 1. 헥사고날 아키텍처 계층별 책임

### Domain 계층
**원칙**: 순수 비즈니스 로직만 포함. 외부 의존성 없음.

| 구성 요소 | 책임 | 포함하는 것 | 포함하지 않는 것 |
|----------|------|-----------|---------------|
| Domain Model | 비즈니스 규칙 캡슐화 | 불변식 검증, 상태 전이, 비즈니스 계산 | DB 접근, 외부 호출, 프레임워크 어노테이션 |
| Enum | 도메인 상수 정의 | 상태값, 타입 구분자 | 인프라 관련 매핑 |

**설계 판단 기준**:
- "이 로직이 DB가 없어도 성립하는가?" → Yes면 Domain 계층
- "이 검증이 비즈니스 규칙인가, 데이터 형식 검증인가?" → 비즈니스 규칙이면 Domain

```kotlin
// 좋은 예: 비즈니스 규칙은 Domain에
class Order(val status: OrderStatus) {
    fun cancel() {
        requireE(status.isCancellable()) { "취소 불가능한 상태입니다" }
        // 비즈니스 규칙: 어떤 상태에서 취소가 가능한지
    }
}

// 나쁜 예: 인프라 관심사가 Domain에 침투
class Order(val status: OrderStatus) {
    @Transactional  // Domain에 프레임워크 의존성
    fun cancel(repository: OrderRepository) { ... }  // 인프라 의존성
}
```

### Application 계층
**원칙**: 비즈니스 흐름 조율(Orchestration). 직접 비즈니스 로직을 구현하지 않음.

| 구성 요소 | 책임 | 설계 근거 |
|----------|------|----------|
| UseCase (Inbound Port) | 외부에서 호출 가능한 비즈니스 연산 정의 | 인터페이스로 정의하여 구현 교체 가능 |
| Port (Outbound Port) | 외부 시스템 의존성 추상화 | 인터페이스로 정의하여 인프라 교체 가능 |
| Service | UseCase 구현, 비즈니스 흐름 조율 | 도메인 모델 호출 + Port 호출 조합 |
| Facade | 여러 Service 간 흐름 조율 | 크로스 도메인 트랜잭션 관리 |
| Handler | 이벤트 수신 및 처리 | 이벤트 기반 비동기 처리 |
| Request/Response DTO | API 계약과 도메인 모델 분리 | 외부 변경이 도메인에 전파되지 않도록 격리 |

**설계 판단 기준**:
- "이 연산이 여러 도메인을 조합하는가?" → Yes면 Facade
- "이 연산이 단일 도메인의 CRUD인가?" → Yes면 Service
- "이 연산이 비동기로 처리되어야 하는가?" → Yes면 Handler

### Infrastructure 계층
**원칙**: Outbound Port 구현. 기술적 세부사항 격리.

| 구성 요소 | 책임 | 설계 근거 |
|----------|------|----------|
| Entity | DB 테이블 매핑 | Domain Model과 1:1이 아닐 수 있음 |
| JpaRepository | Spring Data JPA 기본 CRUD | 단순 조회/저장 |
| QueryRepository | QueryDSL 커스텀 쿼리 | 복잡한 조회, N+1 해결 |
| Adapter | Port 인터페이스 구현 | Domain ↔ Entity 변환 포함 |
| Extension | Domain ↔ Entity 변환 함수 | 확장 함수로 깔끔한 변환 |

---

## 2. CQRS 분리 기준 판단

### 분리가 필요한 경우
- 조회 모델과 명령 모델의 **필드 구성이 크게 다를 때** (조회에 여러 테이블 조인 필요)
- 조회 성능 최적화가 **독립적으로** 필요할 때 (별도 인덱스, 캐싱 전략)
- 명령 후 조회가 **비동기적**으로 반영되어도 괜찮을 때

### 분리하지 않아도 되는 경우
- 단순 CRUD로 조회/명령 모델이 거의 동일할 때
- 조회 빈도가 낮거나 성능 요구사항이 낮을 때

### 분리 시 패키지 구조
```
{domain}/
├── command/
│   ├── inbound/     → UseCase (CUD 연산)
│   ├── outbound/    → Port (저장/수정/삭제)
│   └── {Domain}Service.kt
└── query/
    ├── inbound/     → QueryUseCase (조회 연산)
    ├── outbound/    → QueryPort (조회)
    ├── projection/  → 조회 전용 DTO
    └── Query{Domain}Service.kt
```

---

## 3. Facade 패턴 적용 기준

### 적용하는 경우
- **2개 이상의 도메인 서비스**를 하나의 트랜잭션에서 조율해야 할 때
- 크로스 도메인 비즈니스 흐름을 하나의 진입점으로 제공해야 할 때

### 적용하지 않는 경우
- 단일 도메인 내 단순 위임인 경우 (Service에서 직접 처리)
- 비동기 이벤트로 충분히 분리 가능한 경우 (Handler로 분리)

### Facade vs Event 판단 기준

| 기준 | Facade | Event |
|------|--------|-------|
| 일관성 요구 | 강한 일관성 (즉시 반영 필수) | 최종 일관성 (지연 허용) |
| 실패 처리 | 전체 롤백 필요 | 부분 실패 허용 또는 보상 가능 |
| 결합도 | 동기적 결합 허용 | 느슨한 결합 필요 |

---

## 4. 설계 대안 분석 작성법

TDD에서 "왜 이렇게 설계했는가"를 전달하는 가장 효과적인 방법은 **고려한 대안과 선택 근거**를 함께 제시하는 것이다.

### 작성 원칙
1. **최소 2개 대안**을 비교한다
2. 각 대안의 **장단점을 대칭적으로** 기술한다 (한쪽만 부각하지 않음)
3. 채택 사유에 **구체적 근거**를 제시한다 (성능 수치, 운영 복잡도, 팀 숙련도 등)

### 작성 예시

```markdown
### 설계 대안 분석: 재고 차감 방식

#### 대안 A: 분산 락 (Redisson) - ✅ 채택
- **장점**: 정합성 100% 보장, 구현 패턴이 프로젝트에 이미 존재
- **단점**: 락 대기 시간 발생 (최대 3초), 단일 장애점 (Redis)
- **채택 근거**: 재고 정합성은 금전적 손실과 직결되므로 강한 일관성 필수.
  프로젝트에 Redisson @DistributedLock 패턴이 이미 정착되어 있어 학습 비용 없음.

#### 대안 B: 낙관적 잠금 (@Version)
- **장점**: 락 대기 없음, Redis 의존성 없음
- **단점**: 충돌 시 재시도 로직 필요, 높은 경합 환경에서 재시도 폭증 가능
- **기각 근거**: 프로모션 시 동시 주문이 수백 건 발생하는 시나리오에서
  재시도 폭증이 서비스 장애로 이어질 위험.
```

---

## 5. 처리 흐름 작성법

처리 흐름은 요청이 시스템을 통과하는 전체 경로를 보여준다.
계층 경계를 넘는 지점마다 어떤 인터페이스(Port)를 통해 통신하는지 명시한다.

### 텍스트 기반 시퀀스 표기법

```
[Command 흐름: 주문 취소]

1. Controller: AdminOrderController.cancel(orderId)
   │
2. ├→ UseCase: OrderCancellationUseCase.cancel(request)
   │     - 입력 검증 (Request DTO validation)
   │
3. ├→ Service: OrderCancellationService.cancel(request)
   │     │
4. │    ├→ Port: OrderPort.findById(orderId, tenantId)
   │     │     → 주문 조회 및 존재 확인
   │     │
5. │    ├→ Domain: order.cancel()
   │     │     → 상태 전이 검증 (PAID → CANCEL_REQUESTED)
   │     │     → 비즈니스 규칙 검증
   │     │
6. │    ├→ Port: OrderPort.save(order)
   │     │     → 변경된 주문 저장
   │     │
7. │    └→ Event: OrderCancelledEvent 발행
   │          → @TransactionalEventListener(AFTER_COMMIT)
   │
8. └→ Response: OrderCancellationResponse 반환
```

### 분기가 있는 흐름 표기

```
3. Service: PaymentService.processPayment(request)
   │
   ├─[결제 수단 = CARD]→ PgPort.requestCardPayment(...)
   │                      → PG사 카드 결제 요청
   │
   ├─[결제 수단 = BANK]→ PgPort.requestBankTransfer(...)
   │                      → PG사 계좌이체 요청
   │
   └─[결제 수단 = ISP]→ IspPaymentStrategy.process(...)
                         → ISP 인증 결제 처리
```
