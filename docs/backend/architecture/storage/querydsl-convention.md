# QueryDsl 컨벤션

---

## 핵심 규칙

**동적 조건은 `BooleanExpression?` 반환 방식으로 작성하고, 페이지네이션은 fetchJoin 대신 2-step 조인 방식을 사용한다. Projection은 `Projections.constructor` 기반으로 작성한다.**

쿼리 작성 순서(`select → from → join → where → groupBy → having → orderBy → offset/limit`)를 고정하여 의도가 선언적으로 드러나게 한다. `BooleanBuilder`·`Tuple.get()`·`fetchJoin + 페이지네이션` 같은 우회 방식은 안티패턴이다.

> Adapter · Entity · Extension 작성 규칙은 [storage-adapter-convention.md](storage-adapter-convention.md) 참고

---

## 동적 조건

**규칙: `BooleanBuilder` 대신 `BooleanExpression?` 방식을 사용한다. 단순 조건은 `where` 절에 직접 작성하고, 복잡하거나 재사용되는 조건만 메서드로 추출한다.**

`where()`에 `null`을 전달하면 해당 조건은 무시된다 — QueryDsl의 기본 동작을 활용한다.

### 단순 조건 — where 절에 직접 작성

```kotlin
fun search(condition: FsNodeSearchCondition): List<FsNodeEntity> {
    return queryFactory
        .selectFrom(entity)
        .where(
            condition.name?.let { entity.name.contains(it) },
            condition.type?.let { entity.type.eq(it) },
            condition.minSize?.let { entity.size.goe(it) },
        )
        .fetch()
}
```

### 복잡한 조건 — 메서드 추출

```kotlin
fun search(condition: OrderSearchCondition): List<OrderEntity> {
    return queryFactory
        .selectFrom(order)
        .where(
            condition.status?.let { order.status.eq(it) },
            createdAtBetween(condition.startDate, condition.endDate),
        )
        .fetch()
}

private fun createdAtBetween(start: LocalDate?, end: LocalDate?): BooleanExpression? {
    if (start == null && end == null) return null
    val from = start?.atStartOfDay()
    val to = end?.atTime(LocalTime.MAX)
    return when {
        from != null && to != null -> order.createdAt.between(from, to)
        from != null -> order.createdAt.goe(from)
        else -> order.createdAt.loe(to)
    }
}
```

### 메서드 추출 판단 기준

| 상황 | 방식 |
|------|------|
| 단순 조건 (필드 `eq`, `contains`, `goe` 등) | `where` 절에 직접 작성 |
| 조건 로직이 복잡 (날짜 범위 계산, 다중 OR 조합 등) | `private fun`으로 추출 |
| 여러 Repository에서 동일 조건 재사용 | `private fun`으로 추출 |

### BooleanBuilder를 사용하지 않는 이유

`BooleanBuilder`는 조건을 명령형으로 누적하므로 쿼리의 의도가 분산된다. `where` 절에 직접 나열하면 선언적으로 읽힌다.

```kotlin
// ❌ BooleanBuilder
val builder = BooleanBuilder()
condition.name?.let { builder.and(entity.name.contains(it)) }
condition.type?.let { builder.and(entity.type.eq(it)) }
return queryFactory.selectFrom(entity).where(builder).fetch()
```

---

## 쿼리 작성 규칙

**규칙: 쿼리 작성 순서를 고정하고, 조합 가능성을 기준으로 설계한다.**

### 쿼리 작성 순서

`select → from → join → where → groupBy → having → orderBy → offset/limit` 순서를 고정한다.

```kotlin
fun search(condition: SearchCondition, pageable: Pageable): List<OrderEntity> {
    return queryFactory
        .selectFrom(order)                                     // 1. select/from
        .innerJoin(order.customer, customer).fetchJoin()       // 2. join
        .where(                                                // 3. where
            condition.status?.let { order.status.eq(it) },
            condition.keyword?.let { order.name.contains(it) },
        )
        .orderBy(order.createdAt.desc())                       // 4. orderBy
        .offset(pageable.offset)                               // 5. offset/limit
        .limit(pageable.pageSize.toLong())
        .fetch()
}
```

### 공통 조인 — 확장함수로 묶기

동일한 조인 조합이 여러 쿼리에서 반복되면 `JPAQuery` 확장함수로 추출한다.

```kotlin
private fun <T> JPAQuery<T>.joinOrderRelations(): JPAQuery<T> =
    this.innerJoin(order.customer, customer).fetchJoin()
        .leftJoin(order.items, orderItem).fetchJoin()

fun findDetailById(id: Long): OrderEntity? =
    queryFactory.selectFrom(order).joinOrderRelations().where(order.id.eq(id)).fetchOne()

fun findByIds(ids: List<Long>): List<OrderEntity> =
    queryFactory.selectFrom(order).joinOrderRelations().where(order.id.`in`(ids)).fetch()
```

단일 쿼리에서만 사용하는 조인은 추출하지 않는다 — 인라인이 더 명확하다.

### 조건이 많을 때 — 도메인 의미 기준 그룹핑

`where` 조건이 10~15개 이상이면 도메인 의미 단위로 묶어 `BooleanExpression?` 반환 메서드로 그룹핑한다.

```kotlin
fun search(condition: OrderSearchCondition): List<OrderEntity> {
    return queryFactory
        .selectFrom(order)
        .where(
            orderBasicConditions(condition),
            paymentConditions(condition),
            shippingConditions(condition),
        )
        .fetch()
}

private fun orderBasicConditions(c: OrderSearchCondition): BooleanExpression? {
    val expressions = listOfNotNull(
        c.status?.let { order.status.eq(it) },
        c.orderNumber?.let { order.orderNumber.contains(it) },
        c.customerName?.let { order.customerName.contains(it) },
    )
    return expressions.reduceOrNull { acc, expr -> acc.and(expr) }
}
```

조건이 5개 이하면 그룹핑 없이 `where` 절에 직접 나열한다.

### 재사용 빈도 기준 설계

| 재사용 빈도 | 설계 방침 |
|-------------|----------|
| 단일 쿼리에서만 사용 | `where` 절에 직접 작성하거나 `private fun` |
| 같은 Repository 내 재사용 | `private fun`으로 추출 |
| 여러 Repository에서 재사용 | 별도 유틸 또는 `companion object`로 추출 |

---

## 페이지네이션 — 2-step 조인

**규칙: 페이지네이션에서 fetchJoin을 사용하지 않는다. 2-step 조인 방식을 사용하고, count 쿼리는 content 쿼리와 분리한다.**

fetchJoin + 페이지네이션을 함께 사용하면 Hibernate가 모든 데이터를 메모리에 로드한 후 애플리케이션 레벨에서 페이징을 수행한다 (`HHH90003004` 경고). 데이터가 많아지면 OOM 위험이 있다.

```kotlin
fun search(condition: SearchCondition, pageable: Pageable): Page<Order> {
    // Step 1: 조건에 맞는 ID만 조회 (커버링 인덱스 활용, 가벼운 쿼리)
    val ids = queryFactory
        .select(order.id)
        .from(order)
        .where(statusEq(condition.status), createdAtBetween(condition.startDate, condition.endDate))
        .orderBy(order.createdAt.desc())
        .offset(pageable.offset)
        .limit(pageable.pageSize.toLong())
        .fetch()

    if (ids.isEmpty()) return PageImpl(emptyList(), pageable, 0)

    // Step 2: ID 기준으로 연관 데이터를 fetchJoin (페이징 없이)
    val content = queryFactory
        .selectFrom(order)
        .leftJoin(order.items, orderItem).fetchJoin()
        .leftJoin(order.customer, customer).fetchJoin()
        .where(order.id.`in`(ids))
        .orderBy(order.createdAt.desc())
        .fetch()
        .map { it.toDomain() }

    // Count 쿼리 (content와 분리)
    val total = queryFactory
        .select(order.count())
        .from(order)
        .where(statusEq(condition.status), createdAtBetween(condition.startDate, condition.endDate))
        .fetchOne() ?: 0L

    return PageImpl(content, pageable, total)
}
```

| 단계 | 목적 | 특징 |
|------|------|------|
| Step 1 | ID 목록 조회 | `offset`/`limit` 적용, 커버링 인덱스 활용 |
| Step 2 | 실제 데이터 조회 | `fetchJoin` 사용, `WHERE id IN (...)` |
| Count | 전체 건수 | 조인 없이 조건만으로 count |

전체 건수가 변하지 않는 경우 `PageableExecutionUtils.getPage()`로 count 쿼리를 생략할 수 있다.

---

## Projection

**규칙: 조회 전용 응답에서 Entity 전체가 필요하지 않으면 `Projections.constructor`를 사용한다. `Tuple.get()` 방식은 지양한다.**

```kotlin
data class FsNodeSummary(val id: Long, val name: String, val type: FsNodeType, val size: Long)

fun findSummaries(parentId: Long?): List<FsNodeSummary> {
    return queryFactory
        .select(
            Projections.constructor(
                FsNodeSummary::class.java,
                entity.id, entity.name, entity.type, entity.size,
            )
        )
        .from(entity)
        .where(parentIdEq(parentId))
        .fetch()
}
```

### Tuple.get()을 피하는 이유

`Tuple.get()`은 반환 타입이 `Any?`이므로 컴파일 타임에 타입 오류를 잡을 수 없다. `Projections.constructor`는 DTO 클래스에 직접 매핑하므로 필드 누락이나 타입 불일치를 조기에 발견할 수 있다.

| 방식 | 사용 기준 |
|------|-----------|
| Entity 전체 조회 → `toDomain()` | 도메인 로직이 필요한 경우 (상태 변경, 비즈니스 규칙 적용) |
| `Projections.constructor` | 읽기 전용 목록 조회, 필요한 필드만 선택 |
| `Tuple.get()` | 지양 — 일회성 집계 쿼리 등 극히 제한적인 경우에만 허용 |

---

## 금지 사항

- `BooleanBuilder`를 사용하지 않는다 — `BooleanExpression?` 방식 사용.
- fetchJoin과 페이지네이션을 함께 사용하지 않는다 — 2-step 조인 방식 사용.
- `Tuple.get()` 중심 Projection을 사용하지 않는다 — `Projections.constructor` 사용.
- 쿼리 작성 순서를 임의로 바꾸지 않는다.
- 실제 재사용이 발생하지 않은 조건/조인을 과도하게 추출하지 않는다.

---

## 체크리스트

### 동적 조건
- [ ] `BooleanBuilder` 대신 `BooleanExpression?` 방식을 사용하는가?
- [ ] 단순 조건은 `where` 절에 직접 작성했는가?
- [ ] 복잡하거나 재사용되는 조건만 `private fun`으로 추출했는가?
- [ ] 조건이 10개 이상이면 도메인 의미 기준으로 그룹핑했는가?

### 쿼리 작성
- [ ] 쿼리 작성 순서가 `select → from → join → where → orderBy → offset/limit`인가?
- [ ] 반복되는 조인 조합을 확장함수로 추출했는가? (단일 쿼리는 인라인 유지)

### 페이지네이션
- [ ] fetchJoin과 페이지네이션을 함께 사용하지 않는가?
- [ ] 2-step 조인 방식을 사용하는가?
- [ ] count 쿼리가 content 쿼리와 분리되어 있는가?

### Projection
- [ ] `Tuple.get()` 대신 `Projections.constructor`를 사용하는가?
