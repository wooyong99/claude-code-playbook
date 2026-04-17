# Storage Adapter 컨벤션

---

## 핵심 규칙

**Port 구현체는 `{Entity}Adapter`로 작성하고, JpaRepository와 QueryDslRepository에 위임한다. 모든 반환값은 도메인 객체로 변환하고, JPA Entity는 인프라 계층 밖으로 노출하지 않는다.**

단순 CRUD는 `{Entity}JpaRepository`, 복잡 쿼리는 `{Entity}QueryDslRepository`로 역할을 분리한다. 도메인 ↔ Entity 변환은 `{Entity}Extension.kt`에서만 수행하며 Entity나 Domain 클래스 내부에 변환 로직을 두지 않는다.

> QueryDsl 쿼리 작성 규칙은 [querydsl-convention.md](querydsl-convention.md) 참고

---

## 네이밍 규칙

| 구성 요소 | 패턴 | 예시 |
|-----------|------|------|
| Adapter | `{Entity}Adapter` | `FsNodeAdapter` |
| JpaRepository | `{Entity}JpaRepository` | `FsNodeJpaRepository` |
| QueryDslRepository | `{Entity}QueryDslRepository` | `FsNodeQueryDslRepository` |
| JPA Entity | `{Entity}Entity` | `FsNodeEntity` |
| 변환 파일 | `{Entity}Extension.kt` | `FsNodeExtension.kt` |

---

## Adapter 구조

**규칙: Port 구현체는 `@Repository`로 선언하고, JpaRepository와 QueryDslRepository에 위임한다.**

```kotlin
@Repository
class FsNodeAdapter(
    private val jpaRepository: FsNodeJpaRepository,
    private val queryDslRepository: FsNodeQueryDslRepository,
) : FsNodeRepository {

    override fun save(fsNode: FsNode): FsNode =
        jpaRepository.save(fsNode.toEntity()).toDomain()

    override fun findById(id: Long): FsNode? =
        jpaRepository.findById(id).orElse(null)?.toDomain()

    override fun findAll(): List<FsNode> =
        jpaRepository.findAll().map { it.toDomain() }

    override fun search(condition: FsNodeSearchCondition, pageable: Pageable): Page<FsNode> =
        queryDslRepository.search(condition, pageable)
}
```

- `@Repository`로 선언한다.
- application 계층의 Port 인터페이스를 구현한다.
- 단순 CRUD는 `JpaRepository`, 복잡한 쿼리는 `QueryDslRepository`에 위임한다.
- 모든 반환값은 도메인 객체로 변환한다 — JPA Entity를 외부로 노출하지 않는다.

---

## Repository 분리

**규칙: JpaRepository는 CRUD와 단순 조회, QueryDslRepository는 복잡한 쿼리와 Projection을 담당한다.**

### JpaRepository

```kotlin
interface FsNodeJpaRepository : JpaRepository<FsNodeEntity, Long> {
    fun findByNameAndParentId(name: String, parentId: Long?): FsNodeEntity?
    fun existsByNameAndParentId(name: String, parentId: Long?): Boolean
}
```

| 사용 기준 | 예시 |
|-----------|------|
| 저장 / 삭제 | `save()`, `delete()`, `deleteById()` |
| ID 기반 단건 조회 | `findById()` |
| 단순 조건 조회 (Spring Data 메서드명 쿼리) | `findByNameAndParentId()` |
| 존재 여부 확인 | `existsByNameAndParentId()` |

### QueryDslRepository

| 사용 기준 | 예시 |
|-----------|------|
| 동적 조건 검색 | 검색 필터 조합, 다중 조건 |
| 페이지네이션 + 정렬 | 2-step 조인 페이지네이션 |
| Projection (일부 필드만 조회) | DTO 직접 조회 |
| 복잡한 조인 | 다중 테이블 조인, 서브쿼리 |
| 집계 쿼리 | `groupBy`, `having`, 통계 |

---

## JPA Entity 작성

**규칙: Entity는 순수 JPA 매핑에만 집중한다. 비즈니스 로직을 포함하지 않는다.**

```kotlin
@Entity
@Table(name = "fs_node")
class FsNodeEntity(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0L,

    @Column(nullable = false)
    val name: String,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    val type: FsNodeType,

    val parentId: Long?,

    @Column(nullable = false)
    val size: Long,
)
```

### BaseEntity — 공통 감사(Audit) 필드

생성/수정 시각이 필요한 Entity는 `BaseEntity`를 상속한다.

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseEntity(
    @CreatedDate @Column(updatable = false)
    var createdAt: LocalDateTime = LocalDateTime.now(),
    @LastModifiedDate
    var updatedAt: LocalDateTime = LocalDateTime.now(),
)
```

### Entity 작성 규칙

- `class`로 선언한다 (`data class` 아님 — JPA 프록시 호환).
- `@Column(nullable = false)`로 NOT NULL 제약을 명시한다.
- Enum은 `@Enumerated(EnumType.STRING)`을 사용한다 (순서 변경에 안전).
- 연관관계 매핑은 `@ManyToOne(fetch = LAZY)`를 기본으로 한다.
- `@OneToMany`는 필요한 경우에만 선언한다 (불필요한 양방향 매핑 지양).
- `FetchType.EAGER`를 사용하지 않는다.

---

## 도메인-엔티티 변환

**규칙: 변환 로직은 `{Entity}Extension.kt`에 Extension 함수로 작성한다. Entity나 Domain 클래스 내부에 변환 로직을 두지 않는다.**

변환은 레이어 간 매핑(인프라 관심사)이지, 엔티티나 도메인 객체 자신의 책임이 아니다.

```kotlin
// FsNodeExtension.kt
fun FsNodeEntity.toDomain(): FsNode = FsNode.reconstitute(
    id = id,
    name = name,
    path = path,
    type = type,
    parentId = parentId,
    size = size,
    uploadedAt = uploadedAt,
)

fun FsNode.toEntity(): FsNodeEntity = FsNodeEntity(
    id = id,
    name = name,
    path = path,
    type = type,
    parentId = parentId,
    size = size,
    uploadedAt = uploadedAt,
)
```

- `toDomain()`은 도메인 모델의 `reconstitute()` 팩토리 메서드를 사용한다 (DB 복원이므로 생성 시점 검증 생략).

---

## 금지 사항

- JPA Entity를 인프라 계층 밖으로 반환하지 않는다 — 반드시 `toDomain()`으로 변환.
- Entity에 비즈니스 로직을 두지 않는다.
- Entity나 Domain 클래스 내부에 변환 로직을 두지 않는다 — `{Entity}Extension.kt` 사용.
- `@OneToMany`를 불필요하게 선언하지 않는다.
- `FetchType.EAGER`를 사용하지 않는다.
- `open-in-view: true`를 사용하지 않는다.

---

## 체크리스트

### Adapter
- [ ] 클래스명이 `{Entity}Adapter`인가?
- [ ] `@Repository`로 선언했는가?
- [ ] application 계층의 Port 인터페이스를 구현하는가?
- [ ] 모든 반환값이 도메인 객체인가? (JPA Entity 미노출)

### JpaRepository
- [ ] `JpaRepository<{Entity}Entity, Long>`을 상속하는가?
- [ ] 단순 조회(Spring Data 메서드명 쿼리)만 선언했는가?
- [ ] 복잡한 쿼리는 QueryDslRepository에 위임했는가?

### Entity
- [ ] `class`로 선언했는가? (`data class` 아님)
- [ ] `@Column(nullable = false)`로 NOT NULL 명시했는가?
- [ ] Enum에 `@Enumerated(EnumType.STRING)`을 사용하는가?
- [ ] 연관관계가 `FetchType.LAZY`인가?
- [ ] 감사 필드가 필요하면 `BaseEntity`를 상속하는가?
- [ ] 비즈니스 로직이 없는가?

### 변환
- [ ] `{Entity}Extension.kt`에 `toDomain()` / `toEntity()`가 있는가?
- [ ] `toDomain()`이 도메인 모델의 `reconstitute()`를 사용하는가?
- [ ] Entity나 Domain 클래스 내부에 변환 로직이 없는가?
