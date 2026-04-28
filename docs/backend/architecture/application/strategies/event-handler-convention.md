# Event Handler 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **Event Handler** 역할을 담당하는 `EventHandler` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `DomainEventHandler`, `OutboxProcessor`, `Saga` 등으로 구현할 수 있다.

## 보편 개념

**Domain Event Handler**는 트랜잭션 커밋 이후 발생해야 하는 부수 효과를 처리한다. 이벤트 기반으로 결합하면 트랜잭션 경계를 넘어 다른 도메인에 직접 의존하지 않아도 된다. 이 역할의 컴포넌트는 자기 도메인 책임만 처리하고, 다른 도메인이 필요하면 새 이벤트를 발행하여 해당 도메인의 핸들러가 처리하도록 위임한다.

---

## 원칙

- 이벤트는 비동기 결합의 수단이다. 커밋 후 부수 효과를 이벤트로 처리하면 트랜잭션 경계를 넘어 다른 도메인에 의존하지 않아도 된다.
- 도메인 경계는 이벤트에서도 지킨다. 다른 도메인 Flow를 직접 호출하면 이벤트 기반 분리의 이점이 사라진다.
- EventHandler는 최대한 단순하게 유지한다. 복잡한 조합은 Flow가 담당하고, EventHandler는 자기 도메인 Flow 호출만 수행한다.

---

## 핵심 규칙

**이벤트 핸들러는 자기 도메인의 Flow만 호출한다. 다른 도메인이 필요하면 이벤트를 다시 발행한다.**

`@TransactionalEventListener`로 트랜잭션 커밋 이후에 부수 효과를 처리한다.

---

## 예시

```kotlin
@Component
class OrderEventHandler(
    private val notifyOrderCompleteFlow: NotifyOrderCompleteFlow,
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCompleted(event: OrderCompletedEvent) {
        notifyOrderCompleteFlow.execute(event.orderId)
    }
}
```

---

## 위치

- 이벤트를 처리하는 **개념 영역(도메인)의 패키지 루트**에 flat 배치한다.
- UseCase, EventHandler는 동격으로 도메인 패키지 루트에 위치한다.

```
:core:application/{domain}/
├── {Action}{Entity}UseCase.kt     ← flat 배치
├── {Entity}EventHandler.kt        ← flat 배치 (UseCase와 동격)
└── flow/
    └── {Action}{Entity}Flow.kt
```

---

## 금지 사항

**이벤트 핸들러 안에서 다른 도메인의 Port나 Flow를 직접 호출하지 않는다.** 대신 새 이벤트를 발행하여 해당 도메인의 핸들러가 처리하도록 한다.

```kotlin
// ❌ 다른 도메인 Flow를 직접 호출
@Component
class OrderEventHandler(
    private val notifyFlow: NotifyOrderCompleteFlow,
    private val inventoryDeductFlow: InventoryDeductFlow,  // 다른 도메인
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCompleted(event: OrderCompletedEvent) {
        notifyFlow.execute(event.orderId)
        inventoryDeductFlow.execute(event.orderId)  // ❌ 직접 호출
    }
}

// ✅ 이벤트 재발행
@Component
class OrderEventHandler(
    private val notifyFlow: NotifyOrderCompleteFlow,
    private val applicationEventPublisher: ApplicationEventPublisher,
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCompleted(event: OrderCompletedEvent) {
        notifyFlow.execute(event.orderId)
        applicationEventPublisher.publishEvent(InventoryDeductRequestedEvent(event.orderId))  // ✅ 재발행
    }
}
```

추가 금지 사항:

- `@TransactionalEventListener`의 `phase`를 `AFTER_COMMIT` 외 다른 값으로 임의 변경하지 않는다.
- EventHandler가 UseCase를 호출하지 않는다 — 자기 도메인의 Flow만 호출한다.

---

## 체크리스트

- [ ] `@Component`로 선언했는가?
- [ ] 도메인 패키지 루트에 flat 배치됐는가?
- [ ] `@TransactionalEventListener(phase = AFTER_COMMIT)`을 사용하는가?
- [ ] 자기 도메인의 Flow만 호출하는가?
- [ ] 다른 도메인이 필요한 경우 직접 호출 대신 이벤트를 재발행하는가?
