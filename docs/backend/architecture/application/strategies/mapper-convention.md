# Mapper 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **Assembler** 역할을 담당하는 `Mapper` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `Assembler`, `Converter`, `DtoFactory` 등으로 구현할 수 있다.

## 보편 개념

**Result Assembler**는 Domain 객체를 외부 응답(Result/DTO)으로 변환하는 책임을 하나의 컴포넌트에 집중시킨다. 진입점(UseCase) 내부에 인라인 매핑이 산재하면 변환 규칙을 추적하고 수정하기 어려워진다. 이 역할의 컴포넌트는 순수 변환만 수행하며, 비즈니스 로직과 변환 로직을 섞지 않는다.

---

## 원칙

- 변환 로직은 한 곳에 모인다. UseCase 파일 곳곳에 인라인 매핑이 산재하면 변환 규칙을 추적하고 수정하기 어렵다.
- Mapper는 일관성 원칙을 따른다. 단순한 변환이더라도 Mapper로 분리하면 모든 변환이 예측 가능한 위치에 있다.
- 변환과 비즈니스 로직을 섞지 않는다. Mapper는 순수 변환만 수행하고, 조건 분기가 필요하면 호출 측에서 판단한 뒤 Mapper에 넘긴다.

---

## 핵심 규칙

**도메인 객체를 Result DTO로 변환하는 책임은 Mapper 클래스에 위임한다.**
UseCase에서 직접 인라인 매핑하거나, UseCase 파일 하단에 변환 함수를 추가하지 않는다.

---

## 예시

```kotlin
// mapper/MenuCatalogDtoMapper.kt
@Component
class MenuCatalogDtoMapper {

    fun toCreateResult(menuCatalog: MenuCatalog): CreateMenuCatalog.Result =
        CreateMenuCatalog.Result(
            id = menuCatalog.id,
            parentId = menuCatalog.parentId,
            name = menuCatalog.name,
            path = menuCatalog.path,
            icon = menuCatalog.icon,
            displayOrder = menuCatalog.displayOrder,
            active = menuCatalog.active,
            createdAt = menuCatalog.createdAt,
            updatedAt = menuCatalog.updatedAt,
        )

    fun toGetResult(menuCatalog: MenuCatalog): GetMenuCatalog.Result =
        GetMenuCatalog.Result(
            id = menuCatalog.id,
            parentId = menuCatalog.parentId,
            name = menuCatalog.name,
            path = menuCatalog.path,
            icon = menuCatalog.icon,
            displayOrder = menuCatalog.displayOrder,
            active = menuCatalog.active,
            createdAt = menuCatalog.createdAt,
            updatedAt = menuCatalog.updatedAt,
        )
}
```

---

## 금지 패턴

```kotlin
// ❌ UseCase 내 인라인 매핑
fun create(command: CreateMenuCatalog.Command): CreateMenuCatalog.Result {
    val saved = menuCatalogRepository.save(...)
    return CreateMenuCatalog.Result(id = saved.id, ...)  // 직접 생성 금지
}

// ❌ UseCase 파일 하단에 internal/private 변환 함수 추가
internal fun MenuCatalog.toResult() = CreateMenuCatalog.Result(...)

// ✅ Mapper에 위임
fun create(command: CreateMenuCatalog.Command): CreateMenuCatalog.Result {
    val saved = menuCatalogRepository.save(...)
    return menuCatalogDtoMapper.toCreateResult(saved)
}
```

---

## 추출 판단 기준

- 같은 도메인 객체를 여러 Result 타입으로 변환 → Mapper 하나에서 메서드로 통합
- 여러 도메인 객체를 조합하여 Result 구성 → Mapper
- 단일 UseCase에서만 사용하는 단순 변환이더라도 → Mapper (일관성 원칙)

---

## 체크리스트

- [ ] `@Component`로 선언했는가?
- [ ] `mapper/{Entity}DtoMapper.kt`로 `mapper/` 서브패키지에 위치하는가?
- [ ] UseCase 파일 내부에 인라인 매핑이 남아있지 않는가?
- [ ] UseCase 파일 하단에 `internal`/`private` 변환 함수를 추가하지 않았는가?
