# Domain Model 컨벤션

---

## 핵심 규칙

**도메인 모델은 순수 Kotlin/Java로만 작성하고, `private constructor` + `companion object` 팩토리 메서드로 생성한다. 상태 변경은 행위 메서드로만 허용한다.**

외부 프레임워크(Spring / JPA / Jackson) 변경이 도메인 계층으로 전파되지 않도록 격리하고, `init`·`var` 노출 같은 우회 생성/변경을 차단한다. 외부에서 enum/상태를 꺼내 판단하지 않고 도메인 객체에게 질문한다 (Tell, Don't Ask).

---

## 네이밍 규칙

### 팩토리 메서드

| 메서드명 | 용도 |
|---------|------|
| `create(...)` | 신규 도메인 객체 생성 (불변식 전체 적용) |
| `reconstitute(...)` | 저장된 데이터에서 복원 (생성 시점 규칙 생략 가능) |
| `of(...)` | Value Object 생성 (간결한 표현이 어울릴 때) |
| `from(...)` | 다른 표현에서 변환 (예: `Money.from(rawAmount)`) |

### 도메인 행위 메서드 (Tell, Don't Ask)

| 패턴 | 용도 | 예시 |
|------|------|------|
| `is{State}()` | 현재 상태 확인 | `isActive()`, `isAdmin()` |
| `can{Action}()` | 행위 가능 여부 | `canCancel()`, `canApprove()` |
| `requires{Noun}()` | 정책상 필요 여부 | `requiresAnnualLeave()`, `requiresApproval()` |
| `has{Noun}()` | 보유 여부 | `hasProfileImage()`, `hasPermission()` |

---

## 외부 의존 금지

**규칙: 도메인 계층은 순수 Kotlin 코드만 허용한다. Spring, JPA, Jackson 등 프레임워크에 의존하지 않는다.**

프레임워크에 의존하면 DB 스키마 변경, JSON 포맷 변경 같은 인프라 변화가 도메인 계층 수정을 강제한다. 도메인 모델과 JPA 엔티티를 분리하면 이런 변경이 인프라 계층 내부로 격리된다.

```kotlin
// ❌ 금지
import org.springframework.stereotype.Component
import jakarta.persistence.Entity
import com.fasterxml.jackson.annotation.*

// ✅ 허용
import java.time.LocalDateTime
import java.util.UUID
```

| 의존 대상 | 허용 시 문제 |
|-----------|------------|
| Spring (`@Component`) | 도메인이 Spring 컨테이너 없이 테스트 불가 |
| JPA (`@Entity`) | DB 스키마 변경이 도메인 모델에 전파됨 |
| Jackson (`@JsonProperty`) | JSON 직렬화 방식이 도메인 개념을 오염 |
| HttpStatus | HTTP 프로토콜 개념이 비즈니스 개념과 혼합 |

도메인 모델 ↔ JPA 엔티티 변환은 인프라 계층의 Extension 함수(`toDomain()`, `toEntity()`)가 담당한다.

---

## 도메인 모델 종류

**규칙: Entity는 식별자 기준 동등성을 갖는 `class`로, Value Object는 값 기준 동등성을 갖는 `data class`로 정의한다.**

### Entity

```kotlin
class Order private constructor(
    val id: Long,
    val customerId: Long,
    private var _status: OrderStatus,
    val totalAmount: Long,
) {
    val status: OrderStatus get() = _status

    companion object {
        fun create(customerId: Long, totalAmount: Long): Order {
            require(totalAmount > 0) { "주문 금액은 0보다 커야 합니다." }
            return Order(0L, customerId, OrderStatus.PENDING, totalAmount)
        }

        fun reconstitute(id: Long, customerId: Long, status: OrderStatus, totalAmount: Long): Order =
            Order(id, customerId, status, totalAmount)
    }

    fun confirm() {
        check(_status == OrderStatus.PENDING) { "대기 상태의 주문만 확정할 수 있습니다." }
        _status = OrderStatus.CONFIRMED
    }

    fun cancel() {
        check(_status != OrderStatus.SHIPPED) { "배송 중인 주문은 취소할 수 없습니다." }
        _status = OrderStatus.CANCELLED
    }

    override fun equals(other: Any?) = other is Order && id == other.id
    override fun hashCode() = id.hashCode()
}
```

- 내부 `var`는 허용하되, 반드시 행위 메서드를 통해서만 변경한다.
- 외부에는 `val` getter로만 노출한다 (`private var _status` + `val status get()`).
- `equals` / `hashCode`는 식별자 기준으로 재정의한다.

### Value Object

```kotlin
data class Money private constructor(val amount: Long, val currency: String) {
    companion object {
        fun of(amount: Long, currency: String): Money {
            require(amount >= 0) { "금액은 0 이상이어야 합니다." }
            require(currency.isNotBlank()) { "통화 코드는 비어있을 수 없습니다." }
            return Money(amount, currency)
        }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "통화 단위가 다릅니다." }
        return copy(amount = amount + other.amount)
    }
}
```

---

## 정적 팩토리 메서드

**규칙: `private constructor` + `companion object` 팩토리 메서드로만 생성한다. `init` 블록을 사용하지 않는다.**

```kotlin
class FsNode private constructor(
    val id: Long,
    val tenantId: String,
    val name: String,
    val path: String,
    val type: FsNodeType,
    val parentId: Long?,
    val size: Long,
    val uploadedAt: LocalDateTime,
) {
    companion object {
        fun create(
            tenantId: String,
            name: String,
            path: String,
            type: FsNodeType,
            parentId: Long?,
            size: Long,
        ): FsNode {
            require(tenantId.isNotBlank()) { "테넌트 ID는 비어있을 수 없습니다." }
            require(name.isNotBlank()) { "파일명은 비어있을 수 없습니다." }
            require(size >= 0) { "파일 크기는 0 이상이어야 합니다." }
            return FsNode(0L, tenantId, name, path, type, parentId, size, LocalDateTime.now())
        }

        fun reconstitute(
            id: Long, tenantId: String, name: String, path: String,
            type: FsNodeType, parentId: Long?, size: Long, uploadedAt: LocalDateTime,
        ): FsNode = FsNode(id, tenantId, name, path, type, parentId, size, uploadedAt)
    }
}
```

### 왜 팩토리 메서드인가

1. **의도를 이름으로 표현**: `FsNode.create(...)` vs `FsNode.reconstitute(...)` — 호출부만 읽어도 생성 맥락이 파악된다.
2. **생성 컨텍스트 분리**: `create`는 불변식을 전체 적용하고, `reconstitute`는 DB 복원 시 생성 시점 규칙을 생략할 수 있다.
3. **생성자 직접 노출 차단**: `private constructor`로 검증을 우회하는 생성을 막는다.

### 왜 init을 사용하지 않는가

| 상황 | init의 문제 |
|------|------------|
| `data class`의 `.copy()` | `init`이 재실행 — 의도치 않은 검증 실패 가능 |
| DB 복원 (`reconstitute`) | 과거 데이터가 현재 정책을 만족하지 않을 수 있으나 `init`은 항상 실행 |
| 생성자가 `public` | 팩토리를 거치지 않고 객체 생성 가능 — 검증 우회 |

---

## 불변성 원칙

**규칙: 도메인 객체는 `val`을 기본으로 한다. 상태 변경은 행위 메서드를 통해서만 수행한다.**

`product.price = -1` 같은 직접 변경은 도메인 규칙을 우회한다. 행위 메서드로 감싸면 비즈니스 의도가 드러나고, 검증이 보장된다.

```kotlin
// ❌ 외부에서 직접 변경 가능
class Product(val id: Long, var name: String, var price: Long)

// ✅ 행위 메서드로 상태 변경 표현
class Product private constructor(val id: Long, val name: String, val price: Long) {
    companion object {
        fun create(name: String, price: Long): Product { ... }
        fun reconstitute(id: Long, name: String, price: Long): Product { ... }
    }

    fun rename(newName: String): Product {
        require(newName.isNotBlank()) { "상품명은 비어있을 수 없습니다." }
        return Product(id, newName, price)
    }

    fun adjustPrice(newPrice: Long): Product {
        require(newPrice > 0) { "가격은 0보다 커야 합니다." }
        return Product(id, name, newPrice)
    }
}
```

Entity의 내부 var: 상태 전이가 잦은 Entity는 내부 `var` + 외부 `val` getter 패턴을 사용한다 ("도메인 모델 종류" 섹션의 `Order` 예시 참고).

---

## 도메인 행위 캡슐화 (Tell, Don't Ask)

**규칙: 도메인 객체의 내부 상태를 꺼내서 외부에서 판단하지 않는다. 비즈니스 판단은 도메인 객체에게 질문(메서드 호출)한다.**

외부에서 enum/상태를 직접 비교하면 비즈니스 규칙이 호출부에 분산되고, 정책 변경 시 모든 호출부를 수정해야 하며, 도메인 모델이 데이터 구조체로 전락한다.

### 상태 기반 판단 — 도메인에 캡슐화

```kotlin
// ❌ 외부에서 상태를 꺼내 판단 (Ask)
if (user.userType != UserType.ADMIN) {
    initAnnualLeaveFlow.execute(user.id)
}

// ✅ 도메인 객체에게 질문 (Tell)
if (user.requiresAnnualLeave()) {
    initAnnualLeaveFlow.execute(user.id)
}
```

```kotlin
// User.kt — 비즈니스 규칙을 도메인에 캡슐화
fun requiresAnnualLeave(): Boolean = userType != UserType.ADMIN
```

### enum 직접 비교 vs 도메인 메서드

```kotlin
// ❌ 호출부마다 enum 비교가 산재
if (order.status == OrderStatus.PENDING) { ... }
if (order.status == OrderStatus.PENDING || order.status == OrderStatus.CONFIRMED) { ... }

// ✅ 도메인이 비즈니스 의미를 표현
if (order.canCancel()) { ... }
if (order.isModifiable()) { ... }
```

```kotlin
// Order.kt
fun canCancel(): Boolean = status != OrderStatus.SHIPPED
fun isModifiable(): Boolean = status in setOf(OrderStatus.PENDING, OrderStatus.CONFIRMED)
```

### 판단 기준

| 상황 | 방식 |
|------|------|
| 비즈니스 정책에 따른 분기 (`ADMIN이면 연차 미부여`) | 도메인 메서드 (`requiresAnnualLeave()`) |
| 상태 전이 가능 여부 (`취소 가능한가?`) | 도메인 메서드 (`canCancel()`) |
| 복합 상태 판단 (`수정 가능한 상태인가?`) | 도메인 메서드 (`isModifiable()`) |
| 단순 동등성 비교 (인프라/매핑 계층에서 사용) | 직접 비교 허용 |

---

## 도메인 예외

**규칙: 입력값 전제조건은 `require`, 객체 상태 전제조건은 `check`, 비즈니스 규칙 위반은 `CoreException(errorCode)`를 사용한다.**

```kotlin
// 팩토리 메서드 내 — 입력값 검증
require(name.isNotBlank()) { "상품명은 비어있을 수 없습니다." }

// 행위 메서드 내 — 상태 전제조건
check(status == OrderStatus.PENDING) { "대기 상태의 주문만 확정할 수 있습니다." }

// 비즈니스 규칙 위반 — 핸들링 가능한 도메인 예외
throw CoreException(TenantErrorCode.TENANT_NOT_FOUND)
```

> 예외 계층 구조, ErrorCode enum 설계, 새 예외 추가 방법은 [exception-convention.md](exception-convention.md) 참고

---

## 금지 사항

- Spring, JPA, Jackson 등 외부 프레임워크 import를 사용하지 않는다.
- `init` 블록에서 비즈니스 검증을 수행하지 않는다 — `companion object` 팩토리에서 명시적으로 수행한다.
- `var` 필드를 외부에 직접 노출하지 않는다 — 상태 변경은 행위 메서드로만.
- 생성자를 `public`으로 노출하지 않는다 — `private constructor` + 팩토리 메서드 사용.
- 도메인 모델이 다른 도메인 모델을 직접 포함하지 않는다 — ID 참조로 경계를 유지한다.
- 외부에서 enum/상태를 직접 비교하여 비즈니스 판단을 내리지 않는다 — 도메인 메서드로 캡슐화 (Tell, Don't Ask).

---

## 체크리스트

### 모델 구조
- [ ] `private constructor`를 사용하는가?
- [ ] `companion object`에 `create` / `reconstitute` 팩토리 메서드가 있는가?
- [ ] `init` 블록을 사용하지 않는가?
- [ ] Entity라면 `equals` / `hashCode`를 식별자 기준으로 재정의했는가?
- [ ] Value Object라면 `data class`로 정의했는가?

### 불변성
- [ ] 필드가 `val`로 선언되어 있는가?
- [ ] 상태 변경이 행위 메서드를 통해서만 이루어지는가?
- [ ] 내부 `var`가 필요한 경우 외부에 `val` getter로만 노출하는가?

### 검증
- [ ] 생성 시 입력값 검증을 팩토리 메서드 내에서 `require`로 수행하는가?
- [ ] 상태 전이 전제조건을 행위 메서드 내에서 `check`로 수행하는가?
- [ ] 비즈니스 규칙 위반 시 `CoreException`을 사용하는가?

### 캡슐화 (Tell, Don't Ask)
- [ ] 외부에서 enum/상태 값을 직접 비교하지 않고 도메인 메서드를 호출하는가?
- [ ] 비즈니스 정책 분기(`ADMIN이면 ~`, `PENDING이면 ~`)가 도메인 메서드에 캡슐화되어 있는가?
- [ ] 도메인 메서드명이 비즈니스 의미를 드러내는가? (`isActive`, `canCancel`, `requiresAnnualLeave`)

### 순수성
- [ ] Spring, JPA, Jackson 등 외부 프레임워크 import가 없는가?
- [ ] 순수 Kotlin / Java 표준 라이브러리만 사용하는가?
- [ ] 다른 도메인 모델을 직접 포함하지 않고 ID로 참조하는가?
