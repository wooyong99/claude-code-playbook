# Policy 컨벤션

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

| 구현체 성격 | 위치 |
|------------|------|
| 순수 비즈니스 계산 (외부 의존 없음) | `:core:application` |
| DB/저장소 조회 필요 | `:infra:storage` |
| 내부 시스템 호출 필요 | `:infra:internal` |
| 외부 API 호출 필요 | `:infra:external` |

Spring이 `List<ShippingFeePolicy>`를 주입할 때 모든 모듈의 구현체를 자동으로 포함하므로, Flow는 구현체가 어느 모듈에 있는지 알 필요가 없다.

---

## 추출 판단 기준

| 조건 | 판단 |
|------|------|
| 정책 유형이 2개 이상이고 확장 가능성이 있음 | Policy로 추출 |
| 처리 방식이 2~3가지이고 단순 `when`으로 충분 | Flow 내 `when`으로 유지 |

---

## 체크리스트

- [ ] 인터페이스가 `:core:application`에 정의됐는가?
- [ ] `supports(type)` 디스패치 패턴을 사용하는가?
- [ ] 구현체 모듈 위치가 의존성 기준에 맞는가?
- [ ] `policy/` 서브패키지에 위치하는가?
