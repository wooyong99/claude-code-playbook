# Flow 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **Flow Orchestrator** 역할을 담당하는 `Flow` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `BusinessService`, `Saga`, 또는 진입점 내부 로직으로 통합하여 구현할 수 있다.

## 보편 개념

**Business Flow Orchestrator**는 상태 변경을 동반하는 하나의 업무 흐름을 캡슐화하고, 여러 진입점에서 재사용할 수 있도록 한다. 조회·검증·도메인 행위·저장의 실행 순서를 책임지며, 트랜잭션 경계를 정의한다. 다른 동급 흐름 단위를 직접 호출하지 않는다.

---

## 원칙

- Flow는 하나의 업무 흐름이다. 여러 Port·Validator·Policy의 조합을 단일 흐름으로 캡슐화하고, UseCase가 재사용할 수 있는 단위로 유지한다.
- 상태 변경 없는 흐름은 Flow가 아니다. DB 조회 + 검증만 있다면 ValidateFlow가 아니라 UseCase의 조회 + Validator로 분리하는 것이 올바르다.
- Flow는 자신의 도메인 영역만 안다. 다른 도메인 Port가 필요하면 Handler를 통해 경계를 명확히 한다.
- Flow 간 직접 호출은 UseCase의 역할을 침범한다. 조합이 필요하면 UseCase가 담당한다.

---

## 핵심 규칙

**Flow는 UseCase 아래에서 재사용 가능한 업무 흐름 단위를 담당한다. UseCase는 Flow의 조합만 결정한다.**

Flow는 Validator · Handler · Policy · Port를 조합하여 하나의 업무 흐름을 완성한다.
비즈니스 로직의 실체가 담기는 곳이다.

---

## 네이밍 규칙

**규칙: Flow는 `{Entity}{Action}Flow` 형식으로 네이밍한다. UseCase(`{Action}{Entity}UseCase`)와 반대 순서다.**

- **UseCase**: `{Action}{Entity}UseCase` — `CreateProductUseCase`, `DeleteProductUseCase`
- **Flow**: `{Entity}{Action}Flow` — `ProductCreateFlow`, `ProductDeleteFlow`

UseCase는 Action이 먼저, Flow는 Entity가 먼저다 (`CreateAccountFlow` / `DeleteProductFlow` 처럼 Action 먼저 쓰면 UseCase 패턴과 혼동됨).

---

## Flow 메서드 내부 처리 순서

1. 데이터 조회 — Port
2. 비즈니스 규칙 검증 — Validator (조회된 객체를 매개변수로 전달)
3. 외부 영역 위임 — Handler
4. 정책 결정 — Policy (행위 분기, 필요 시)
5. 도메인 행위 호출 — Domain 객체 메서드
6. 저장 — Port.save()
7. 결과 반환 — 도메인 객체 반환

---

## 예시

### 기본 Flow

```kotlin
@Component
class FileUploadFlow(
    private val fileStorage: FileStorage,
    private val fsNodeRepository: FsNodeRepository,
    private val fileValidator: FileValidator,
) {
    @Transactional
    fun execute(tenant: Tenant, command: UploadFile.Command): FsNode {
        val existing = fsNodeRepository.findByNameAndParent(command.fileName, command.parentId)

        fileValidator.validateUploadPermission(tenant, command.fileName)
        fileValidator.validateNoDuplicate(existing, command.fileName)

        val path = fileStorage.saveFile(command.fileName, command.bytes)
        val fsNode = FsNode(
            name = command.fileName, path = path,
            type = FsNodeType.FILE, parentId = command.parentId,
            size = command.bytes.size.toLong(),
        )
        return fsNodeRepository.save(fsNode)
    }
}
```

### 여러 Flow를 조합하는 UseCase

```kotlin
// @Transactional 없음 → Flow 커밋 후 외부 API 호출
@Service
class CreateOrderUseCase(
    private val tenantRepository: TenantRepository,
    private val orderPersistFlow: OrderPersistFlow,
    private val paymentExternalExecutor: PaymentExecutor,
    private val orderMapper: OrderDtoMapper,
) {
    fun create(command: CreateOrder.Command): CreateOrder.Result {
        val tenant = tenantRepository.findById(command.tenantId)
            ?: throw CoreException(TenantErrorCode.TENANT_NOT_FOUND)

        val order = orderPersistFlow.execute(tenant, command)  // 트랜잭션 커밋

        paymentExternalExecutor.request(order)  // DB 커넥션 반납 후 호출

        return orderMapper.toResult(order)
    }
}
```

### 여러 Flow를 하나의 트랜잭션으로 묶어야 하는 경우

```kotlin
@Service
class TransferFundsUseCase(
    private val debitFlow: DebitFlow,
    private val creditFlow: CreditFlow,
    private val transferMapper: TransferDtoMapper,
) {
    @Transactional  // 출금과 입금이 반드시 하나의 트랜잭션이어야 함
    fun transfer(command: TransferFunds.Command): TransferFunds.Result {
        val debit = debitFlow.execute(command)
        val credit = creditFlow.execute(command)
        return transferMapper.toResult(debit, credit)
    }
}
```

---

## 추출 판단 기준

- 두 개 이상의 UseCase에서 동일한 실행 흐름이 필요 → Flow로 추출
- 여러 Port · Validator · Handler를 조합하는 복잡한 흐름 → Flow로 추출
- UseCase 하나에서만 사용하는 단순 흐름 → UseCase 내 private 메서드로 유지
- 단일 Port 호출만을 위한 흐름 → Flow 생성 금지. UseCase에서 Port 직접 호출

### Flow 내부 처리 vs UseCase에서 묶기 판단

"해당 흐름을 빼면 나머지만으로 비즈니스가 성립하는가?"

- 빼면 불완전 → 단일 비즈니스 오퍼레이션 (예: 부서 생성 시 Role 생성) → Flow 내부에서 Handler를 통해 함께 처리
- 빼도 각각 성립 → 독립적인 오퍼레이션 조합 (예: 사용자 등록 + 역할 배정 + 연차 초기화) → UseCase에서 `@Transactional`로 여러 Flow 묶기

---

## 트랜잭션 경계

- **UseCase**: 선언하지 않는 것이 원칙. 여러 Flow를 하나의 트랜잭션으로 묶어야 할 때만 예외
- **Flow**: 쓰기 작업이 있으면 `@Transactional` 필수. 읽기만 있으면 `@Transactional(readOnly = true)`

**외부 API 호출**: UseCase에 `@Transactional`이 없으면 Flow 커밋 후 ExternalExecutor를 호출할 수 있어
DB 커넥션을 불필요하게 오래 점유하지 않는다.
하나의 트랜잭션으로 묶어야 하는 경우, `@TransactionalEventListener(phase = AFTER_COMMIT)`을 활용한다.

---

## 금지 사항

- Flow에서 다른 Flow를 직접 호출하지 않는다 — Flow 조합은 UseCase가 담당한다.
- Flow에서 다른 개념 영역의 Port를 직접 주입하지 않는다 — 해당 영역의 Handler에 위임한다.
- 단일 Port 호출만을 위해 Flow를 만들지 않는다.
- **검증만을 위한 Flow(`*ValidateFlow`)를 만들지 않는다** — 아래 안티패턴 참고.

---

## ValidateFlow 안티패턴

Flow의 핵심 목적은 **상태 변경을 동반하는 업무 흐름**이다. DB 조회 + 비즈니스 규칙 검증만 수행하고 상태 변경이 없는 컴포넌트는 Flow가 아니다.

```
❌ RequiredTermsValidateFlow   — 상태 변경 없음. Validator와 경계 혼란.
```

이런 컴포넌트가 Port 주입을 필요로 하는 이유는 "검증에 필요한 데이터를 직접 조회하기 때문"이다.
올바른 해결책은 **조회 책임을 호출 측(UseCase/Flow)으로 올리고 Validator를 순수하게 유지**하는 것이다.

```kotlin
// ❌ 안티패턴 — ValidateFlow
@Component
class RequiredTermsValidateFlow(
    private val termsRepository: TermsRepository,  // Port 주입으로 Validator 규칙 위반
) {
    @Transactional(readOnly = true)
    fun execute(agreedTermsVersionIds: List<Long>) {
        val requiredIds = termsRepository.findActiveRequiredVersions()...
        // 조회 + 검증이 혼재
    }
}

// ✅ 올바른 분리 — UseCase에서 조회, Validator에서 검증
@Service
class RegisterTenantUseCase(
    private val termsRepository: TermsRepository,       // UseCase에서 직접 조회
    private val termsRequiredValidator: TermsRequiredValidator,
    // ...
) {
    @Transactional
    fun register(command: RegisterTenant.Command) {
        val requiredVersionIds = termsRepository.findActiveRequiredVersions()
            .map { it.version.id }.toSet()
        val activeVersions = if (command.agreedTermsVersionIds.isNotEmpty())
            termsRepository.findActiveVersionsByIds(command.agreedTermsVersionIds.toSet())
        else emptyList()

        // 조회된 데이터를 Validator에 전달 — 검증 규칙만 Validator가 담당
        termsRequiredValidator.validate(
            requiredVersionIds = requiredVersionIds,
            agreedVersionIds = command.agreedTermsVersionIds.toSet(),
            activeVersions = activeVersions,
        )

        // 이후 Flow 조합...
    }
}
```

**전제 조건을 관련 없는 도메인 Flow 안에 넣지 않는다.** 예를 들어 "약관 동의 검증"은 Tenant 생성의 도메인 불변식이 아니므로 `TenantCreateFlow` 안에 두지 않는다. `RegisterTenantUseCase`의 선행 조건으로 UseCase 레벨에서 처리한다.

> 검증 위치(UseCase vs Flow) 판단 기준 상세는 [validator-convention.md](validator-convention.md) "Validator 호출 위치" 섹션 참고

---

## 체크리스트

- [ ] `@Component`로 선언했는가?
- [ ] 쓰기는 `@Transactional`, 읽기는 `@Transactional(readOnly = true)`가 선언됐는가?
- [ ] Flow가 다른 Flow를 직접 호출하지 않는가?
- [ ] 다른 개념 영역의 Port를 직접 주입하지 않고 Handler를 통해 접근하는가?
- [ ] 단일 Port 호출만을 위한 Flow가 만들어지지 않았는가?
- [ ] 외부 API 호출(ExternalExecutor)을 트랜잭션 안에서 하지 않는가?
- [ ] 상태 변경 없이 검증만 수행하는 Flow(`*ValidateFlow`)를 만들지 않았는가?
