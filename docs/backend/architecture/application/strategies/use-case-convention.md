# UseCase 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **Entry Point** 역할을 담당하는 `UseCase` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `Service`, `CommandHandler`, `ApplicationService` 등으로 구현할 수 있다.

## 보편 개념

**Application Entry Point**는 외부(HTTP, 이벤트, CLI 등)의 요청을 받아 업무 흐름을 조립하는 진입점이다. "무엇을 어떤 순서로 실행할지"를 결정하고, 실행 자체는 하위 컴포넌트에 위임한다. 이 역할의 컴포넌트는 다른 동급 진입점을 직접 호출하지 않으며, 비즈니스 로직을 직접 구현하지 않는다.

---

## 원칙

- UseCase는 "무엇을 조합할지"만 결정한다. 비즈니스 로직이 UseCase에 들어오면 재사용 단위인 Flow가 의미를 잃는다.
- 트랜잭션 경계는 Flow에서 관리하는 것이 원칙이다. UseCase가 트랜잭션을 선언하면 DB 커넥션을 불필요하게 오래 점유할 수 있다.
- 하나의 개념 = 하나의 UseCase. 행위 유형(조회/쓰기)으로 나누되, 개념이 다르면 반드시 분리한다.
- Command의 검증 범위는 형식에 한정한다. DB 조회가 필요한 규칙은 Command init에 두지 않고 Validator/Flow에 위임한다.

---

## 핵심 규칙

**UseCase는 `@Service`로 선언하고 Flow의 조합만 결정한다. 비즈니스 로직은 Flow에 위임한다.**

트랜잭션은 원칙적으로 Flow에서 관리한다. UseCase는 여러 Flow를 하나의 트랜잭션으로 묶어야 할 때만 예외적으로 `@Transactional`을 선언한다.

지원 컴포넌트 상세:
[flow-convention.md](flow-convention.md) · [event-handler-convention.md](event-handler-convention.md) · [validator-convention.md](validator-convention.md) · [handler-convention.md](handler-convention.md) · [policy-convention.md](policy-convention.md) · [mapper-convention.md](mapper-convention.md)

---

## 네이밍 규칙

- **클래스명**: `{Action}{Entity}UseCase` 패턴 (행위가 먼저, 개념이 뒤) — `GetProductUseCase`, `UploadFileUseCase`, `CreateOrderUseCase`
- **메서드명**: `execute` 대신 도메인 행위명 — `create`, `get`, `getList`, `getDetail`, `update`, `delete`, `upload`

---

## UseCase 메서드 구성

**UseCase 메서드 — 조합 순서:**

1. 공통 데이터 조회 — Port (여러 Flow에서 공유할 상위 객체)
2. Flow 호출 — Flow (실행 로직 위임)
3. 결과 반환 — Mapper

> Flow 메서드 내부 처리 순서는 [flow-convention.md](flow-convention.md) "Flow 메서드 내부 처리 순서" 섹션 참고

### Command init에 넣는 것 / 넣지 않는 것

```kotlin
class UploadFile {
    data class Command(
        val name: String,
        val bytes: ByteArray,
        val parentId: Long?,
    ) {
        init {
            require(name.isNotBlank()) { "파일명은 비어있을 수 없습니다." }
            require(bytes.isNotEmpty()) { "파일 내용이 없습니다." }
        }
    }
}
```

- ✅ 넣는 것: 필수값 존재 여부, 값 범위, 이메일 패턴, 최대 길이
- ❌ 넣지 않는 것: 허용 확장자 검사 → **Validator**, 엔티티 존재 여부 → **Port 조회 후 Validator에 전달**

---

## 예시

```kotlin
@Service
class UploadFileUseCase(
    private val tenantRepository: TenantRepository,
    private val fileUploadFlow: FileUploadFlow,
    private val fsNodeMapper: FsNodeDtoMapper,
) {
    fun upload(command: UploadFile.Command): UploadFile.Result {
        val tenant = tenantRepository.findById(command.tenantId)
            ?: throw CoreException(TenantErrorCode.TENANT_NOT_FOUND)

        val fsNode = fileUploadFlow.execute(tenant, command)

        return fsNodeMapper.toResult(fsNode)
    }
}
```

### 하나의 개념 = 하나의 UseCase

동일한 개념 객체를 다루는 여러 행위는 **하나의 UseCase에 메서드 단위로** 통합한다.

```kotlin
@Service
class GetFsNodeUseCase(...) {
    fun getList(): GetFsNodeList.Result { ... }
    fun getDetail(id: Long): GetFsNodeDetail.Result { ... }
}

@Service
class UpdateFsNodeUseCase(...) {
    fun rename(command: RenameFsNode.Command): RenameFsNode.Result { ... }
    fun move(command: MoveFsNode.Command): MoveFsNode.Result { ... }
}
```

- 같은 개념, 같은 행위 유형 (조회/수정) → 하나의 UseCase, 메서드 분리
- 같은 개념, 다른 행위 유형 (조회 vs 수정) → 별도 UseCase
- 다른 개념 객체 → 반드시 별도 UseCase

---

## 의존성 방향

**규칙: 애플리케이션 계층 내부는 아래 방향으로만 의존한다. 역방향 금지.**

```
UseCase → Flow → Validator / Handler / Policy → Port (interface)
```

- **UseCase**: Flow, Port (단순 조회), Domain 의존 가능 / 다른 UseCase 의존 불가
- **Flow**: Validator, Handler, Policy, Port, Domain 의존 가능 / UseCase, 다른 Flow 의존 불가
- **Validator**: Domain 의존 가능 / Port, Flow, UseCase 의존 불가
- **Handler**: Port, Domain 의존 가능 / Flow, UseCase 의존 불가
- **Policy 인터페이스**: `:core:application`에 정의
- **Policy 구현체**: 의존성에 따라 `:core:application` / `:infra:*`에 위치
- **Port**: interface — 의존 없음

> **Flow → Flow 금지**: 여러 Flow를 조합해야 하는 경우는 UseCase가 담당한다.

---

## Command/Result 파일 네이밍

**규칙: Command와 Result는 operation 단위 파일 하나에 함께 정의한다. `{Action}{Entity}.kt` 파일에 nested class로 둔다.**

```kotlin
// UploadFile.kt
class UploadFile {
    data class Command(val tenantId: String, val name: String, val bytes: ByteArray, val parentId: Long?) {
        init {
            require(name.isNotBlank()) { "파일명은 비어있을 수 없습니다." }
            require(bytes.isNotEmpty()) { "파일 내용이 없습니다." }
        }
    }
    data class Result(val id: Long, val name: String, val path: String)
}

// 사용부
fun upload(command: UploadFile.Command): UploadFile.Result
```

Command가 없는 조회는 Result만 정의한다:

```kotlin
// GetFsNodeDetail.kt
class GetFsNodeDetail {
    data class Result(val id: Long, val name: String, val path: String, val type: FsNodeType)
}
```

sealed class 타입 분기:

```kotlin
// UpdateProduct.kt
class UpdateProduct {
    sealed class Command {
        data class UpdateInfo(val id: Long, val name: String, val price: Long) : Command()
        data class UpdateImage(val id: Long, val imageBytes: ByteArray) : Command()
    }
    data class Result(val id: Long, val name: String, val price: Long)
}
```

- **파일명**: `{Action}{Entity}.kt` (항상 단수) — `UploadFile.kt`, `CreateOrder.kt`
- **Wrapper 클래스명**: 파일명과 동일 — `class UploadFile { ... }`
- **Command**: `{Wrapper}.Command` — `UploadFile.Command`
- **Result**: `{Wrapper}.Result` — `UploadFile.Result`
- **Command 없는 조회**: Result만 정의 — `GetFsNodeDetail.Result`

---

## 금지 사항

- `BaseUseCase` 같은 공통 상속 클래스를 만들지 않는다.
- UseCase가 다른 UseCase를 직접 호출하지 않는다.
- Command `init`에서 DB 조회가 필요한 검증을 하지 않는다.
- UseCase 파일 하단에 `internal`/`private` 변환 함수를 추가하지 않는다 — Mapper 사용 ([mapper-convention.md](mapper-convention.md) 참고).

> Flow 내부 관련 금지 사항은 [flow-convention.md](flow-convention.md) "금지 사항" 섹션 참고

---

## 체크리스트

- [ ] `@Service`로 선언했는가?
- [ ] 클래스명이 `{Action}{Entity}UseCase` 형식인가?
- [ ] 메서드명이 도메인 행위명(`create`, `upload`, `getDetail` 등)을 사용하는가?
- [ ] `BaseUseCase` 같은 공통 상속 클래스를 상속하지 않는가?
- [ ] `@Transactional`을 UseCase에 선언하지 않았는가? (여러 Flow를 하나의 트랜잭션으로 묶는 경우 제외)
- [ ] 비즈니스 로직을 직접 구현하지 않고 Flow에 위임했는가?
- [ ] 다른 UseCase를 직접 호출하지 않는가?
- [ ] 외부 API 호출(ExternalExecutor)을 Flow 커밋 이후에 하고 있는가?
- [ ] 파일명이 `{Action}{Entity}.kt` 형식이고 Command/Result가 nested class로 정의됐는가?
- [ ] Command의 `init`에서 형식 검증만 하고 DB 조회가 없는가?
- [ ] UseCase → Flow → Validator/Handler/Policy → Port 방향만 의존하는가?
