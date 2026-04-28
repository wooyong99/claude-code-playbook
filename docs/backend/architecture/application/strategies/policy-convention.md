# Policy 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **Strategy** 역할을 담당하는 `Policy` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `Strategy`, `Rule`, `Specification` 인터페이스로 구현할 수 있다.

## 보편 개념

**Strategy Pattern**은 확장 가능성이 있는 행위 분기를 인터페이스로 외부화하여, 신규 전략 추가 시 기존 코드를 수정하지 않도록 한다. 분기 판단 책임을 호출 측이 아닌 전략 구현체 자신이 갖도록(`supports()` 메서드) 호출 코드를 일관되게 유지한다. 구현체의 위치는 의존성이 결정한다.

---

## 원칙

- Policy는 분기 전략을 외부화한다. `when`으로 처리하기엔 확장 가능성이 있는 행위 분기를 인터페이스로 격리하면 신규 정책 추가 시 기존 코드를 수정하지 않아도 된다.
- Policy 구현체의 위치는 의존성이 결정한다. 순수 계산은 application, DB 조회가 필요하면 infra 모듈에 위치시킨다.
- `List<Policy>` + `supports()` 패턴을 통일한다. 분기를 호출 측이 아닌 Policy 자신이 판단하게 하여 호출 코드를 일관되게 유지한다.

---

## 핵심 규칙

**정책에 따라 처리 방식이 달라지는 지점을 캡슐화한다.**
인터페이스는 `:core:application`에 정의하고, 구현체는 의존성에 따라 배치한다.
`List<Policy>` 주입 + `supports(type)` 디스패치 패턴을 사용한다.

---

## 예시

```kotlin
// 인터페이스 — :core:application
interface ShippingFeePolicy {
    fun supports(type: ShippingType): Boolean
    fun calculate(orderAmount: Long): Long
}

// 구현체
@Component
class StandardShippingPolicy : ShippingFeePolicy {
    override fun supports(type: ShippingType) = type == ShippingType.STANDARD
    override fun calculate(orderAmount: Long) = if (orderAmount >= 50_000) 0L else 3_000L
}
```

Flow에서 `List<Policy>` 주입 + `supports(type)` 디스패치:

```kotlin
@Component
class OrderPersistFlow(
    private val shippingPolicies: List<ShippingFeePolicy>,
    private val orderRepository: OrderRepository,
) {
    @Transactional
    fun execute(command: CreateOrder.Command): Order {
        val policy = shippingPolicies
            .firstOrNull { it.supports(command.shippingType) }
            ?: throw CoreException(OrderErrorCode.INVALID_ORDER)

        val shippingFee = policy.calculate(command.orderAmount)
        val order = Order.create(command.customerId, command.orderAmount, shippingFee)
        return orderRepository.save(order)
    }
}
```

---

## 구현체 모듈 위치

- 순수 비즈니스 계산 (외부 의존 없음) → `:core:application`
- DB/저장소 조회 필요 → `:infra:storage`
- 내부 시스템 호출 필요 → `:infra:internal`
- 외부 API 호출 필요 → `:infra:external`

Spring이 `List<ShippingFeePolicy>`를 주입할 때 모든 모듈의 구현체를 자동으로 포함하므로, Flow는 구현체가 어느 모듈에 있는지 알 필요가 없다.

---

## 추출 판단 기준

- 정책 유형이 2개 이상이고 확장 가능성이 있음 → Policy로 추출
- 처리 방식이 2~3가지이고 단순 `when`으로 충분 → Flow 내 `when`으로 유지

---

## 체크리스트

- [ ] 인터페이스가 `:core:application`에 정의됐는가?
- [ ] `supports(type)` 디스패치 패턴을 사용하는가?
- [ ] 구현체 모듈 위치가 의존성 기준에 맞는가?
- [ ] `policy/` 서브패키지에 위치하는가?
