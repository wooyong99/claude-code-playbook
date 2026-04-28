# Handler 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **ACL / Coordinator** 역할을 담당하는 `Handler` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `Facade`, `ApplicationService`, `DomainService` 등으로 구현할 수 있다.

## 보편 개념

**Anti-Corruption Layer / Cross-domain Coordinator**는 두 가지 역할을 담당한다. 첫째, 여러 도메인 영역에서 반복되는 공통 로직을 하나의 재사용 단위로 추출한다. 둘째, 한 도메인의 흐름 단위가 다른 도메인의 인프라에 직접 접근하는 것을 차단하여 도메인 간 결합을 방지한다. 단순 위임이나 단일 Port 호출만을 위해 이 역할의 컴포넌트를 만들지 않는다.

---

## 원칙

- Handler는 도메인 간 경계를 보호한다. Flow가 다른 도메인에 직접 접근하면 도메인 간 결합이 생기고, Handler가 그 경계를 차단하는 ACL 역할을 한다.
- 재사용이 확인된 로직만 Handler로 추출한다. 한 Flow에서만 사용하는 로직을 Handler로 올리면 불필요한 간접 계층이 생긴다.
- 단일 Port 호출은 Handler를 거치지 않는다. 단순한 위임에 Handler를 만들면 코드 경로만 길어진다.

---

## 핵심 규칙

**여러 도메인에서 재사용되는 로직은 Handler로 추출한다. 단일 Port 호출이라면 Flow에서 직접 호출한다.**

Handler는 Flow가 다른 개념 영역에 접근할 때 ACL(Anti-Corruption Layer) 역할을 한다.
다른 도메인 Port의 변경이 Flow에 전파되지 않도록 경계를 차단한다.

---

## 예시

```kotlin
@Component
class AttachedFileHandler(
    private val fileStorage: FileStorage,
    private val fsNodeRepository: FsNodeRepository,
) {
    fun replace(targetId: Long, name: String, bytes: ByteArray): FsNode { ... }
    fun store(name: String, bytes: ByteArray): FsNode { ... }
    fun delete(fsNodeId: Long) { ... }
}
```

UseCase에서의 사용:

```kotlin
@Service
class UpdateProductUseCase(
    private val productRepository: ProductRepository,
    private val attachedFileHandler: AttachedFileHandler,
    private val productMapper: ProductDtoMapper,
) {
    fun updateImage(command: UpdateProductImage.Command): UpdateProductImage.Result {
        val product = productRepository.findById(command.productId)
            ?: throw CoreException(TenantErrorCode.TENANT_NOT_FOUND)

        val fsNode = attachedFileHandler.replace(
            targetId = product.id, name = command.fileName, bytes = command.bytes,
        )

        product.updateImagePath(fsNode.path)
        productRepository.save(product)
        return productMapper.toResult(product)
    }
}
```

---

## 추출 판단 기준

- 여러 도메인에서 재사용, 여러 Port/Service 조합 → Handler로 추출
- Flow에서 다른 개념 영역 Port 접근 필요 → Handler로 추출 (ACL)
- 여러 도메인에서 재사용, 단일 Port 호출 → Flow에서 직접 호출
- Flow 하나에서만 사용 → Flow의 private 메서드로 유지

---

## 금지 사항

- Flow 하나에서만 사용되는 로직을 Handler로 추출하지 않는다 — Flow의 `private` 메서드로 유지한다.
- 여러 도메인에서 재사용되더라도 **단일 Port 호출**만 필요한 경우 Handler를 만들지 않는다 — Flow에서 직접 호출한다.
- Handler가 UseCase나 Flow를 호출하지 않는다 — 의존 방향은 Handler → Port / Domain 까지만 허용된다.

---

## 체크리스트

- [ ] `@Component`로 선언했는가?
- [ ] 해당 개념 도메인 패키지 루트에 flat 배치됐는가?
- [ ] Flow 하나에서만 사용되는 로직을 불필요하게 Handler로 추출하지 않았는가?
- [ ] Flow에서 다른 개념 영역 Port를 직접 주입하지 않고 이 Handler를 통해 접근하는가?
