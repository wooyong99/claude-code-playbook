# 도메인 모델링 참조 가이드

이 문서는 TDD 6단계(도메인 모델링 및 데이터 설계) 작성 시 참조하는 상세 가이드다.

---

## 목차
1. [애그리거트 경계 판단 기준](#1-애그리거트-경계-판단-기준)
2. [ID 참조 패턴](#2-id-참조-패턴)
3. [도메인 모델 상세 작성법](#3-도메인-모델-상세-작성법)
4. [상태 전이 설계](#4-상태-전이-설계)
5. [스냅샷 패턴](#5-스냅샷-패턴)
6. [DDL 작성 규칙](#6-ddl-작성-규칙)
7. [Extension 함수 변환 패턴](#7-extension-함수-변환-패턴)

---

## 1. 애그리거트 경계 판단 기준

애그리거트는 "함께 변경되어야 하는 객체들의 묶음"이다.
경계를 잘못 잡으면 트랜잭션이 비대해지거나, 정합성이 깨진다.

### 판단 질문

| 질문 | Yes → | No → |
|------|-------|------|
| A 없이 B가 존재할 수 있는가? | 별도 애그리거트 | 같은 애그리거트 |
| A와 B를 항상 함께 조회/수정하는가? | 같은 애그리거트 고려 | 별도 애그리거트 |
| A 수정 시 B의 불변식이 깨질 수 있는가? | 같은 트랜잭션 필요 | 분리 가능 |
| A와 B의 생명주기가 동일한가? | 같은 애그리거트 고려 | 별도 애그리거트 |

### 프로젝트 기존 패턴 참고

```
[Product 애그리거트] ← 하나의 트랜잭션 단위
├── Product (루트)
├── ProductPrice
├── ProductOption
├── ProductOptionValue
└── ProductImage

[Order 애그리거트]
├── Order (루트)
├── OrderItem
└── OrderPayment

[별도 애그리거트 - ID 참조]
├── OrderItemProductSnapshot  ← Order에서 productId로 참조
└── OrderCouponSnapshot       ← Order에서 couponId로 참조
```

### 경계 결정 시 문서화 형식

```markdown
### 애그리거트 경계

#### [NewDomain] 애그리거트 (루트: NewDomain)
- **구성**: NewDomain, NewDomainDetail, NewDomainOption
- **경계 근거**: NewDomainDetail은 NewDomain 없이 존재할 수 없으며,
  항상 함께 생성/수정/삭제된다. 하나의 트랜잭션으로 관리해야 불변식을 보장할 수 있다.

#### [ExistingDomain]과의 관계: ID 참조
- **참조 방식**: NewDomain.existingDomainId (Long)
- **분리 근거**: ExistingDomain은 독립적 생명주기를 가지며,
  NewDomain 생성 시점과 무관하게 변경될 수 있다.
```

---

## 2. ID 참조 패턴

이 프로젝트는 **JPA 연관관계(@OneToMany, @ManyToOne)를 사용하지 않는다**.
모든 애그리거트 간 참조는 ID 값으로만 이루어진다.

### 원칙
- 같은 애그리거트 내부: 객체 직접 참조 가능 (Domain 모델 내)
- 다른 애그리거트 간: **ID 값 참조만 허용**
- 데이터 조합: Infrastructure 계층의 QueryRepository에서 QueryDSL로 수행

### 설계 문서 작성 시 표기법

```
[주문 도메인 관계도]

Order (1) ──→ (N) OrderItem          [orderId로 참조]
OrderItem ──→ ProductVariant          [productVariantId로 참조 - 별도 애그리거트]
OrderItem (1) ──→ (1) ProductSnapshot [orderItemId로 참조]
Order ──→ Member                      [memberId로 참조 - 별도 애그리거트]
```

### 데이터 조합 패턴

ID만으로 참조하므로, 조회 시 데이터 조합은 Infrastructure 계층에서 수행한다.

```kotlin
// QueryDSL transform을 사용한 효율적 조합
fun findOrderDetail(orderId: Long, tenantId: Long): OrderDetailProjection {
    return queryFactory
        .from(orderEntity)
        .leftJoin(orderItemEntity).on(orderItemEntity.orderId.eq(orderEntity.id))
        .where(
            orderEntity.id.eq(orderId),
            orderEntity.tenantId.eq(tenantId)
        )
        .transform(
            groupBy(orderEntity.id).`as`(
                QOrderDetailProjection(
                    orderEntity.id,
                    // ... order fields
                    list(QOrderItemProjection(
                        orderItemEntity.id,
                        // ... item fields
                    ))
                )
            )
        )[orderId] ?: throw EEntityNotFoundException("주문을 찾을 수 없습니다")
}
```

---

## 3. 도메인 모델 상세 작성법

각 도메인 클래스에 대해 다음 항목을 문서화한다.

### 작성 템플릿

```markdown
#### [ClassName]

**역할**: [이 클래스가 담당하는 비즈니스 규칙을 한 문장으로]

**불변식 (Invariants)**:
- `productName`은 빈 문자열일 수 없다 → `requireE(productName.isNotBlank())`
- `price`는 0 이상이어야 한다 → `requireE(price >= BigDecimal.ZERO)`
- `status`가 DELETED이면 수정 불가 → `checkE(status != Status.DELETED)`

**주요 행위**:
| 메서드 | 비즈니스 의미 | 사전 조건 | 사후 조건 |
|--------|-------------|----------|----------|
| `create(...)` | 신규 생성 | 필수 필드 충족 | 초기 상태 = DRAFT |
| `activate()` | 활성화 | status = DRAFT | status = ACTIVE |
| `delete()` | 논리 삭제 | status ≠ DELETED | deleteYn = true |

**필드 정의**:
| 필드 | 타입 | nullable | 비즈니스 의미 |
|------|------|----------|-------------|
| id | Long | No | PK |
| tenantId | Long | No | 테넌트 식별자 |
| status | StatusEnum | No | 현재 상태 |
```

### 불변식 코드화 패턴

이 프로젝트에서 불변식은 `requireE`(전제조건), `checkE`(상태검증), `entityNotNull`(존재검증)로 표현한다.

```kotlin
class Product(
    val tenantId: Long,
    productName: String,
    val status: ProductStatus = ProductStatus.DRAFT
) {
    var productName = productName
        private set

    init {
        validate(productName = productName)
    }

    private fun validate(productName: String? = null) {
        productName?.let {
            requireE(it.isNotBlank()) { "product.name.required".toI18n() }
            requireE(it.length <= 250) { "product.name.max_250".toI18n() }
        }
    }

    fun updateName(newName: String) {
        checkE(status != ProductStatus.DELETED) { "삭제된 상품은 수정할 수 없습니다" }
        validate(productName = newName)
        this.productName = newName
    }
}
```

---

## 4. 상태 전이 설계

상태를 가진 도메인은 **허용되는 전이 경로**를 명시적으로 정의한다.

### 상태 전이 다이어그램 표기법

```
[OrderStatus 상태 전이]

PENDING ──→ PAID ──→ PREPARING ──→ SHIPPED ──→ DELIVERED
   │          │                                    │
   │          ├──→ CANCEL_REQUESTED ──→ CANCELLED  │
   │          │                                    │
   └──→ CANCELLED                    RETURN_REQUESTED ──→ RETURNED
         (결제 전 취소)                  (배송 후 반품)
```

### 상태 전이 매트릭스 (권장)

| 현재 상태 \ 이벤트 | 결제 완료 | 취소 요청 | 배송 시작 | 배송 완료 | 반품 요청 |
|-------------------|----------|----------|----------|----------|----------|
| PENDING | → PAID | → CANCELLED | - | - | - |
| PAID | - | → CANCEL_REQ | → PREPARING | - | - |
| PREPARING | - | - | → SHIPPED | - | - |
| SHIPPED | - | - | - | → DELIVERED | - |
| DELIVERED | - | - | - | - | → RETURN_REQ |

### Enum에서 전이 검증 패턴

```kotlin
enum class OrderStatus {
    PENDING, PAID, PREPARING, SHIPPED, DELIVERED, CANCELLED;

    fun canTransitionTo(next: OrderStatus): Boolean = when (this) {
        PENDING -> next in setOf(PAID, CANCELLED)
        PAID -> next in setOf(PREPARING, CANCELLED)
        PREPARING -> next == SHIPPED
        SHIPPED -> next == DELIVERED
        DELIVERED -> false
        CANCELLED -> false
    }
}
```

---

## 5. 스냅샷 패턴

주문 시점의 데이터를 보존하여 향후 참조 데이터 변경이 과거 주문에 영향을 주지 않도록 하는 패턴.

### 적용 판단 기준

| 질문 | Yes → | No → |
|------|-------|------|
| 참조 데이터가 변경될 수 있는가? | 스냅샷 고려 | ID 참조로 충분 |
| 변경 시 과거 기록의 정합성이 깨지는가? | 스냅샷 필수 | ID 참조 |
| 법적/회계적으로 시점 데이터 보존이 필요한가? | 스냅샷 필수 | 상황에 따라 |

### 프로젝트 기존 스냅샷 예시
- `order_item_product_snapshots`: 주문 시점 상품명, 가격, 옵션 정보
- `order_coupon_snapshots`: 주문 시점 쿠폰 정보와 할인 금액

### 설계 문서 작성 시

스냅샷을 적용할 때 다음을 명시한다:
1. **어떤 데이터를 스냅샷하는가**: 복사 대상 필드 목록
2. **왜 스냅샷이 필요한가**: 원본 변경 시 발생하는 문제
3. **스냅샷 시점**: 어떤 시점에 스냅샷을 생성하는가
4. **원본과의 관계**: originalId로 원본 추적 가능 여부

---

## 6. DDL 작성 규칙

이 프로젝트는 DDL-First 원칙을 따른다. 설계 문서의 스키마는 SQL DDL 형태로 작성한다.

### DDL 템플릿

```sql
CREATE TABLE {table_name} (
    id              BIGINT          NOT NULL AUTO_INCREMENT,
    tenant_id       BIGINT          NOT NULL,
    -- 비즈니스 필드
    {field_name}    {TYPE}          {NULL/NOT NULL} {DEFAULT},
    -- 상태 필드
    status          VARCHAR(50)     NOT NULL DEFAULT 'DRAFT',
    delete_yn       TINYINT(1)      NOT NULL DEFAULT 0,
    -- 감사 필드
    created_at      DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at      DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    created_by      VARCHAR(255)    NULL,
    updated_by      VARCHAR(255)    NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 필수 규칙
- **모든 테이블에 `tenant_id` 포함** (멀티테넌트 격리)
- **논리 삭제 사용**: `delete_yn TINYINT(1) DEFAULT 0`
- **감사 필드 포함**: `created_at`, `updated_at`, `created_by`, `updated_by`
- **ENUM은 VARCHAR로 매핑**: MySQL ENUM 대신 `VARCHAR + @Enumerated(EnumType.STRING)`
- **외래 키 제약 조건 사용하지 않음**: ID 참조 패턴과 일관성 유지

### 인덱스 설계 가이드
```sql
-- 복합 인덱스: 테넌트 ID를 선두에 배치 (멀티테넌트 쿼리 최적화)
CREATE INDEX idx_{table}_{columns} ON {table_name} (tenant_id, {column1}, {column2});

-- 유니크 제약: 비즈니스 유니크 + 테넌트 격리
CREATE UNIQUE INDEX uk_{table}_{columns} ON {table_name} (tenant_id, {unique_column});
```

---

## 7. Extension 함수 변환 패턴

Domain ↔ Entity ↔ DTO 변환은 Extension 함수로 구현한다.

### 변환 경로

```
Request DTO → Domain Model → Entity
  (application/extension/)    (infrastructure/extension/)

Entity → Domain Model → Response DTO
  (infrastructure/extension/)  (application/extension/)
```

### 설계 문서에서 변환 흐름 표기

```markdown
### 데이터 변환 흐름

| 방향 | 변환 | 위치 | 설명 |
|------|------|------|------|
| 생성 | Request → Domain | application/.../extension/ | 사용자 입력 → 비즈니스 모델 |
| 저장 | Domain → Entity | infrastructure/.../extension/ | 비즈니스 모델 → DB 매핑 |
| 조회 | Entity → Domain | infrastructure/.../extension/ | DB 매핑 → 비즈니스 모델 |
| 응답 | Domain → Response | application/.../extension/ | 비즈니스 모델 → API 응답 |
| 직접 조회 | Entity → Projection | QueryRepository | QueryDSL Projection 직접 매핑 |
```

조회 최적화가 필요한 경우, Entity → Domain 변환 없이 QueryDSL Projection으로 직접 조회 결과를 매핑한다.
이 경우 Projection 클래스는 `query/projection/` 패키지에 위치한다.
