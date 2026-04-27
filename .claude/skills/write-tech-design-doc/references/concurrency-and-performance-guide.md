# 동시성 제어 및 성능 참조 가이드

이 문서는 TDD 9단계(동시성 제어 및 성능 고려사항) 작성 시 참조하는 상세 가이드다.

---

## 목차
1. [동시성 제어 방식 선택 기준](#1-동시성-제어-방식-선택-기준)
2. [분산 락 설계 상세](#2-분산-락-설계-상세)
3. [낙관적 잠금 패턴](#3-낙관적-잠금-패턴)
4. [N+1 문제 해결 패턴](#4-n1-문제-해결-패턴)
5. [캐싱 전략 설계](#5-캐싱-전략-설계)
6. [확장 가능성 문서화](#6-확장-가능성-문서화)
7. [TDD 동시성/성능 섹션 작성 예시](#7-tdd-동시성성능-섹션-작성-예시)

---

## 1. 동시성 제어 방식 선택 기준

### 방식별 비교

| 방식 | 경합 빈도 | 정합성 | 성능 영향 | 구현 복잡도 |
|------|----------|--------|----------|-----------|
| 분산 락 (Redisson) | 높은 경합 | 강한 일관성 | 락 대기 발생 | 중간 (기존 패턴 있음) |
| 낙관적 잠금 (@Version) | 낮은 경합 | 충돌 시 재시도 | 재시도 비용 | 낮음 |
| 비관적 잠금 (SELECT FOR UPDATE) | 중간 경합 | 강한 일관성 | DB 락 대기 | 낮음 |
| 큐 기반 직렬화 | 매우 높은 경합 | 순서 보장 | 지연 발생 | 높음 |

### 선택 판단 플로우

```
동시 쓰기가 발생하는가?
├── No → 동시성 제어 불필요
└── Yes → 멀티 인스턴스 환경인가?
    ├── No → DB 비관적 잠금으로 충분
    └── Yes → 경합 빈도는?
        ├── 낮음 (초당 10건 미만) → 낙관적 잠금 + 재시도
        └── 높음 (초당 10건 이상) → 분산 락 (Redisson)
            └── 매우 높음 (초당 100건+) → 분산 락 + 큐 기반 버퍼링 고려
```

---

## 2. 분산 락 설계 상세

### @DistributedLock 어노테이션

```kotlin
@DistributedLock(
    key = "stock:#request.productVariantId:#tenantId",
    waitTime = 3,     // 락 획득 대기 최대 시간 (초)
    leaseTime = 10    // 락 자동 해제 시간 (초) - 데드락 방지
)
fun decreaseStock(request: StockRequest.Decrease) {
    // 이 메서드는 동시에 하나의 스레드만 실행
}
```

### 락 키 설계 원칙

| 원칙 | 설명 | 예시 |
|------|------|------|
| 세분화 | 경합 범위를 최소화 | `stock:{variantId}` (O) / `stock:{productId}` (X) |
| 테넌트 격리 | 테넌트 간 경합 방지 | `stock:{variantId}:{tenantId}` |
| 의미 부여 | 키만 보고 어떤 자원인지 파악 | `order-cancel:{orderId}:{tenantId}` |

### waitTime / leaseTime 설정 가이드

| 연산 특성 | waitTime | leaseTime | 근거 |
|----------|----------|-----------|------|
| 빠른 연산 (재고 차감) | 3초 | 10초 | 처리 1초 이내, 여유 확보 |
| 중간 연산 (주문 생성) | 5초 | 30초 | 다단계 처리, 외부 호출 가능 |
| 느린 연산 (결제 처리) | 10초 | 60초 | PG 응답 대기 포함 |

### 분산 락 실패 시 처리

```kotlin
// 락 획득 실패 시 DistributedLockException 발생
// → 클라이언트에 "잠시 후 재시도" 안내
```

---

## 3. 낙관적 잠금 패턴

경합이 드물 때 사용하는 방식. 충돌 발생 시 재시도한다.

### JPA @Version 사용

```kotlin
@Entity
class ProductEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    // ... fields
    @Version
    val version: Long = 0  // 자동 증가, 충돌 시 OptimisticLockingFailureException
)
```

### 낙관적 잠금 + 재시도

```kotlin
@Retryable(
    value = [OptimisticLockingFailureException::class],
    maxAttempts = 3,
    backoff = Backoff(delay = 100, multiplier = 2.0)  // 100ms → 200ms → 400ms
)
fun updateProductName(productId: Long, newName: String) {
    val product = productPort.findById(productId)
    product.updateName(newName)
    productPort.save(product)
}
```

### TDD에서 낙관적 잠금 기술 시

재시도 횟수 초과 시 처리 전략을 반드시 명시한다.

---

## 4. N+1 문제 해결 패턴

이 프로젝트에서 1:N 관계 조회 시 N+1 문제는 QueryDSL `transform(groupBy)` 패턴으로 해결한다.

### 패턴: 단일 쿼리 + 인메모리 그룹핑

```kotlin
fun findOrderDetail(orderId: Long, tenantId: Long): OrderDetailProjection {
    return queryFactory
        .from(orderEntity)
        .leftJoin(orderItemEntity).on(
            orderItemEntity.orderId.eq(orderEntity.id),
            orderItemEntity.tenantId.eq(tenantId)
        )
        .where(
            orderEntity.id.eq(orderId),
            orderEntity.tenantId.eq(tenantId)
        )
        .transform(
            groupBy(orderEntity.id).`as`(
                QOrderDetailProjection(
                    orderEntity.id,
                    orderEntity.orderNumber,
                    // ... order fields
                    list(QOrderItemProjection(
                        orderItemEntity.id,
                        orderItemEntity.productVariantId,
                        // ... item fields
                    ))
                )
            )
        )[orderId] ?: throw EEntityNotFoundException("주문을 찾을 수 없습니다")
}
```

### 패턴: 다단계 조합 (3개 이상 1:N 관계)

```kotlin
fun findProductDetail(productId: Long, tenantId: Long): ProductDetailProjection {
    // 1. 메인 데이터 조회
    val product = fetchProduct(productId, tenantId)

    // 2. 1:N 데이터 각각 조회 (별도 쿼리)
    val options = fetchProductOptions(productId, tenantId)
    val images = fetchProductImages(productId, tenantId)
    val variants = fetchProductVariants(productId, tenantId)

    // 3. 인메모리 조합
    return product.copy(
        options = options,
        images = images,
        variants = variants
    )
}
```

### TDD에서 쿼리 성능 기술 시

| 조회 연산 | 예상 쿼리 수 | 해결 패턴 | 설명 |
|----------|------------|----------|------|
| 주문 상세 조회 | 1 (join + groupBy) | transform 패턴 | order + items 단일 쿼리 |
| 상품 목록 조회 | 1 + N (categories) | 2단계 조합 | 상품 → 카테고리 별도 조회 |
| 주문 목록 (페이지) | 2 (content + count) | PageableExecutionUtils | 카운트 쿼리 지연 실행 |

---

## 5. 캐싱 전략 설계

이 프로젝트는 Redis 기반 캐싱을 사용한다.

### 캐시 적용 판단 기준

| 조건 | 캐시 적용 | 근거 |
|------|----------|------|
| 자주 조회, 드물게 변경 | 적합 | 히트율 높음 |
| 테넌트별 독립 데이터 | 적합 (키에 tenantId 포함) | 격리 보장 |
| 실시간 정합성 필수 | 부적합 | 캐시 무효화 지연 위험 |
| 대량 데이터 | 선별 적용 | 메모리 효율 고려 |

### 캐시 어노테이션 패턴

```kotlin
// 조회 시 캐시 적용
@Cacheable(
    cacheNames = ["skinByShopAndType"],
    key = "'shop:' + #shopId + ':type:' + #skinType.name()"
)
fun findByShopIdAndSkinType(shopId: Long, skinType: SkinType): Skin

// 저장/수정 시 캐시 무효화
@CacheEvict(
    cacheNames = ["skinByShopAndType"],
    key = "'shop:' + #skin.shopId + ':type:' + #skin.skinType.name()"
)
fun save(skin: Skin): Skin

// 복수 엔티티 저장 시 프로그래밍 방식 무효화
fun saveAll(skins: List<Skin>): List<Skin> {
    val result = repository.saveAll(skins.map { it.toEntity() })
    val cache = cacheManager.getCache("skinByShopAndType")
    skins.forEach { skin ->
        cache?.evict("shop:${skin.shopId}:type:${skin.skinType.name}")
    }
    return result.map { it.toDomain() }
}
```

### TDD에서 캐시 전략 기술 시

| 대상 | 캐시명 | 키 패턴 | TTL | 무효화 조건 | 사유 |
|------|--------|---------|-----|-----------|------|
| 스킨 조회 | skinByShopAndType | shop:{shopId}:type:{type} | 1h | 스킨 저장/삭제 시 | 변경 빈도 낮음, 조회 빈번 |

---

## 6. 확장 가능성 문서화

TDD의 확장 가능성 섹션에서는 현재 설계가 열어둔 **확장 포인트**와 의도적으로 닫아둔 **제약**을 모두 서술한다.

### 작성 형식

```markdown
### 7.3 확장 가능성

#### 열린 확장 포인트
| 확장 시나리오 | 현재 설계의 대응 | 필요 변경량 |
|-------------|----------------|-----------|
| 새로운 결제 수단 추가 | PaymentStrategy 인터페이스 | 전략 클래스 1개 추가 |
| 새로운 CS 유형 추가 | CsType enum + 별도 상세 테이블 | enum 값 + 테이블 + 서비스 추가 |

#### 의도적 제약
| 제약 | 사유 | 해제 조건 |
|------|------|----------|
| 단일 PG사만 지원 | 현재 요구사항 범위 | 멀티 PG 요구 시 Strategy 패턴 확장 |
| 동기 처리만 지원 | 처리량 충분 | TPS 1000+ 시 이벤트 기반 전환 검토 |
```

### 작성 원칙
- "만약에 대비한" 과도한 추상화를 권장하지 않는다 (YAGNI)
- 대신 "이 지점이 확장 포인트"라고 명시하여, 나중에 확장이 필요할 때 어디를 건드려야 하는지 안내한다
- 현재 설계의 한계를 솔직하게 인정하되, 해제 조건을 함께 기술한다

---

## 7. TDD 동시성/성능 섹션 작성 예시

```markdown
## 7. 동시성 및 성능

### 7.1 동시성 제어

| 경합 지점 | 제어 방식 | 구현 | waitTime | leaseTime | 사유 |
|----------|----------|------|----------|-----------|------|
| 재고 차감 | 분산 락 | @DistributedLock(key="stock:{variantId}:{tenantId}") | 3s | 10s | 프로모션 시 동시 주문 100건+ 대응, 정합성 최우선 |
| 주문 취소 | 분산 락 | @DistributedLock(key="order-cancel:{orderId}:{tenantId}") | 3s | 10s | 동일 주문 동시 취소 방지 |
| 상품 정보 수정 | 낙관적 잠금 | @Version | - | - | 관리자 동시 수정 드묾, 충돌 시 재시도 |

### 7.2 성능 고려사항

| 항목 | 우려 사항 | 대응 전략 | 측정 기준 |
|------|----------|----------|----------|
| 주문 상세 조회 | OrderItem N+1 | transform(groupBy) 패턴 | 쿼리 1회로 해결 |
| 주문 목록 페이지 | 카운트 쿼리 비용 | PageableExecutionUtils 지연 실행 | 전체 카운트 불필요 시 스킵 |
| 스킨 조회 | 매 요청 DB 조회 | Redis 캐시 (TTL 1h) | 히트율 95% 이상 목표 |

### 7.3 확장 가능성

#### 열린 확장 포인트
| 확장 시나리오 | 현재 설계의 대응 | 필요 변경량 |
|-------------|----------------|-----------|
| 부분 취소 지원 | OrderItem 단위 취소 가능하도록 설계 | Service + 상태 전이 추가 |
| 자동 환불 재시도 | 이벤트 핸들러에 재시도 로직 | @Retryable 추가 |

#### 의도적 제약
| 제약 | 사유 | 해제 조건 |
|------|------|----------|
| 환불은 비동기 처리 | PG 응답 지연으로 동기 처리 시 UX 저하 | 즉시 환불 PG API 제공 시 |
| 단일 통화만 지원 | 국내 서비스 한정 | 해외 진출 시 통화 관리 도메인 추가 |
```
