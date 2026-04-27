# 동시성 제어 및 성능 정책

백엔드 서비스의 동시성 제어 방식 선택 기준과 쿼리 성능·캐시·확장성에 관한 공통 정책.

---

## 목차
1. [동시성 제어 방식 선택 기준](#1-동시성-제어-방식-선택-기준)
2. [분산 락 설계 상세](#2-분산-락-설계-상세)
3. [낙관적 잠금 패턴](#3-낙관적-잠금-패턴)
4. [N+1 문제 해결 패턴](#4-n1-문제-해결-패턴)
5. [캐싱 전략 설계](#5-캐싱-전략-설계)
6. [확장 가능성 문서화](#6-확장-가능성-문서화)

---

## 1. 동시성 제어 방식 선택 기준

### 방식별 비교

| 방식 | 경합 빈도 | 정합성 | 성능 영향 | 구현 복잡도 |
|------|----------|--------|----------|-----------|
| 분산 락 | 높은 경합 | 강한 일관성 | 락 대기 발생 | 중간 |
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
        └── 높음 (초당 10건 이상) → 분산 락
            └── 매우 높음 (초당 100건+) → 분산 락 + 큐 기반 버퍼링 고려
```

---

## 2. 분산 락 설계 상세

### 사용 패턴

```kotlin
@DistributedLock(
    key = "stock:{productVariantId}",
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
| 리소스 격리 | 충분한 식별자로 의도치 않은 경합 방지 | `stock:{variantId}:{scopeId}` |
| 의미 부여 | 키만 보고 어떤 자원인지 파악 가능 | `order-cancel:{orderId}` |

### waitTime / leaseTime 설정 가이드

| 연산 특성 | waitTime | leaseTime | 근거 |
|----------|----------|-----------|------|
| 빠른 연산 | 3초 | 10초 | 처리 1초 이내, 여유 확보 |
| 중간 연산 | 5초 | 30초 | 다단계 처리, 외부 호출 가능 |
| 느린 연산 | 10초 | 60초 | 외부 시스템 응답 대기 포함 |

락 획득 실패 시 클라이언트에 "잠시 후 재시도" 안내를 반환한다.

---

## 3. 낙관적 잠금 패턴

경합이 드물 때 사용하는 방식. 충돌 발생 시 재시도한다.

### JPA @Version 사용

```kotlin
@Entity
class ProductEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
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

재시도 횟수 초과 시 처리 전략(예외 전파, fallback)을 명시한다.

---

## 4. N+1 문제 해결 패턴

1:N 관계 조회 시 N+1 문제는 단일 쿼리 + 인메모리 그룹핑 패턴으로 해결한다.

### 패턴: 단일 쿼리 + 인메모리 그룹핑

```kotlin
fun findOrderDetail(orderId: Long): OrderDetailProjection {
    return queryFactory
        .from(orderEntity)
        .leftJoin(orderItemEntity).on(orderItemEntity.orderId.eq(orderEntity.id))
        .where(orderEntity.id.eq(orderId))
        .transform(
            groupBy(orderEntity.id).`as`(
                QOrderDetailProjection(
                    orderEntity.id,
                    orderEntity.orderNumber,
                    list(QOrderItemProjection(
                        orderItemEntity.id,
                        orderItemEntity.productVariantId,
                    ))
                )
            )
        )[orderId] ?: throw EntityNotFoundException("엔티티를 찾을 수 없습니다")
}
```

### 패턴: 다단계 조합 (3개 이상 1:N 관계)

```kotlin
fun findProductDetail(productId: Long): ProductDetailProjection {
    // 1. 메인 데이터 조회
    val product = fetchProduct(productId)

    // 2. 1:N 데이터 각각 별도 쿼리로 조회
    val options = fetchProductOptions(productId)
    val images = fetchProductImages(productId)
    val variants = fetchProductVariants(productId)

    // 3. 인메모리 조합
    return product.copy(options = options, images = images, variants = variants)
}
```

### 조회 성능 설계 시 고려 항목

| 조회 연산 | 예상 쿼리 수 | 해결 패턴 | 설명 |
|----------|------------|----------|------|
| 상세 조회 (1:N 단일) | 1 (join + groupBy) | 인메모리 그룹핑 | 단일 쿼리로 N+1 방지 |
| 상세 조회 (1:N 복수) | 2단계 분리 | 다단계 조합 | 별도 쿼리 후 메모리 결합 |
| 페이지 목록 | 2 (content + count) | 지연 카운트 실행 | 카운트 쿼리 불필요 시 스킵 |

---

## 5. 캐싱 전략 설계

### 캐시 적용 판단 기준

| 조건 | 캐시 적용 | 근거 |
|------|----------|------|
| 자주 조회, 드물게 변경 | 적합 | 히트율 높음 |
| 리소스별 독립 데이터 | 적합 (키에 식별자 포함) | 격리 보장 |
| 실시간 정합성 필수 | 부적합 | 캐시 무효화 지연 위험 |
| 대량 데이터 | 선별 적용 | 메모리 효율 고려 |

### 캐시 어노테이션 패턴

```kotlin
// 조회 시 캐시 적용
@Cacheable(
    cacheNames = ["productByType"],
    key = "'product:' + #productId + ':type:' + #productType.name()"
)
fun findByProductIdAndType(productId: Long, productType: ProductType): Product

// 저장/수정 시 캐시 무효화
@CacheEvict(
    cacheNames = ["productByType"],
    key = "'product:' + #product.id + ':type:' + #product.type.name()"
)
fun save(product: Product): Product

// 복수 엔티티 저장 시 프로그래밍 방식 무효화
fun saveAll(products: List<Product>): List<Product> {
    val result = repository.saveAll(products.map { it.toEntity() })
    val cache = cacheManager.getCache("productByType")
    products.forEach { product ->
        cache?.evict("product:${product.id}:type:${product.type.name}")
    }
    return result.map { it.toDomain() }
}
```

### 캐시 전략 설계 시 포함할 항목

| 항목 | 설명 |
|------|------|
| 캐시명 | 용도를 명시하는 의미 있는 이름 |
| 키 패턴 | 충분히 세분화된 캐시 키 (충돌 방지) |
| TTL | 데이터 변경 빈도에 맞는 유효 기간 |
| 무효화 조건 | 언제 캐시를 제거할지 |
| 적용 근거 | 왜 캐시가 필요한지 |

---

## 6. 확장 가능성 문서화

설계의 **확장 포인트**와 의도적으로 닫아둔 **제약**을 함께 문서화한다.

### 열린 확장 포인트 작성 형식

| 확장 시나리오 | 현재 설계의 대응 | 필요 변경량 |
|-------------|----------------|-----------|
| 새로운 처리 전략 추가 | Strategy 인터페이스 | 전략 클래스 1개 추가 |
| 새로운 타입 추가 | enum + 상세 테이블 | enum 값 + 테이블 + 서비스 추가 |

### 의도적 제약 작성 형식

| 제약 | 사유 | 해제 조건 |
|------|------|----------|
| 동기 처리만 지원 | 현재 처리량 충분 | TPS 임계치 초과 시 이벤트 기반 전환 검토 |
| 단일 외부 시스템만 지원 | 현재 요구사항 범위 | 멀티 시스템 요구 시 Strategy 패턴 확장 |

### 작성 원칙
- "만약에 대비한" 과도한 추상화를 권장하지 않는다 (YAGNI)
- "이 지점이 확장 포인트"라고 명시하여, 나중에 확장이 필요할 때 어디를 건드려야 하는지 안내한다
- 현재 설계의 한계를 솔직하게 인정하되, 해제 조건을 함께 기술한다
