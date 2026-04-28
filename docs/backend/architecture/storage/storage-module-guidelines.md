# Storage Layer Guidelines

## Storage 계층의 본질적 책임

`:infra:storage` 모듈은 application 계층의 Port 인터페이스를 구현하여 도메인 객체를 영속화한다. DB 스키마 변경이 도메인 계층으로 전파되지 않도록 JPA Entity와 도메인 모델을 항상 분리한다.

1. **Port 구현**: application 계층이 선언한 저장소 인터페이스(Port)를 Adapter로 구현한다.
2. **도메인 격리**: JPA Entity는 인프라 경계 안에 머무르며, 반환 전 반드시 도메인 객체로 변환한다.
3. **조회 복잡도 분리**: 단순 CRUD는 `JpaRepository`, 동적 조건·복잡 조인은 `QueryDslRepository`로 역할을 나눈다.

```
application (Port 인터페이스)
  ↑ 구현
storage  — Adapter → JpaRepository / QueryDslRepository → DB
             ↕ (변환)
          Entity ↔ Domain (Extension 함수)
```

---

## 반드시 지켜야 할 규칙

- **R1. JPA Entity를 인프라 계층 밖으로 노출하지 않는다** — Adapter의 모든 반환값은 `toDomain()`으로 변환한다. Entity를 그대로 반환하면 도메인 모델과 인프라 모델이 결합된다.
- **R2. 변환은 `{Entity}Extension.kt`에서만 수행한다** — `toDomain()` / `toEntity()` 변환 로직을 Entity나 Domain 클래스 내부에 두지 않는다.
- **R3. Repository 역할을 분리한다** — 단순 CRUD·단순 조건 조회는 `JpaRepository`, 동적 조건·페이지네이션·Projection·복잡 조인은 `QueryDslRepository`가 담당한다.

---

## 금지 규칙 / 안티패턴

- **JPA Entity 계층 외부 노출** — Entity를 application이나 domain 계층으로 반환하면 인프라 스키마 변경이 상위 계층에 전파된다.
- **Entity 내 비즈니스 로직** — JPA Entity는 DB 매핑 전용 클래스이며 도메인 규칙을 포함해서는 안 된다.
- **Entity·Domain 클래스 내 변환 로직** — 변환 책임이 여러 클래스에 분산되면 변경 시 모두 찾아야 한다.
- **JpaRepository에 복잡한 쿼리** — `@Query`로 QueryDsl이 필요한 쿼리를 JpaRepository에 구현하면 역할 경계가 흐려진다.

---

## 이 프로젝트의 로컬 컨벤션

### 공통 규칙

**파일 구조**: `StorageConfig.kt`(설정) + `common/`(공통 컴포넌트) + `{domain}/`(도메인별 컴포넌트) 구조를 따른다. `basePackages`는 루트 패키지로 고정되어 하위 패키지 추가 시 별도 설정 변경이 필요하지 않다.

```
:infra:storage/
├── StorageConfig.kt                  ← @EntityScan, @EnableJpaRepositories (basePackages 고정)
├── common/
│   └── BaseEntity.kt                 ← 공통 감사(Audit) 필드 (@CreatedDate, @LastModifiedDate)
└── {domain}/
    ├── {Entity}Entity.kt             ← JPA Entity
    ├── {Entity}JpaRepository.kt      ← Spring Data JPA (CRUD / 단순 조회)
    ├── {Entity}QueryDslRepository.kt ← QueryDsl (복잡 쿼리 / Projection)
    ├── {Entity}Adapter.kt            ← Port 구현체
    └── {Entity}Extension.kt          ← toDomain() / toEntity()
```

**테스트**: Adapter · Repository 테스트는 `@DataJpaTest` 또는 `@SpringBootTest`로 실제 DB 연동 테스트한다. QueryDsl 동적 조건과 페이지네이션(2-step 조인)은 통합 테스트로 검증한다.

**DDL**: 엔티티 신규·변경 시 DDL 파일을 **같은 커밋**에서 함께 갱신한다. → [ddl-management.md](ddl-management.md) 참고

### Post-Work Verification

구현 완료 후 생성·수정한 파일을 직접 읽어 아래 각 문서의 체크리스트를 대조한다.

- [storage-adapter-convention.md](storage-adapter-convention.md)
- [querydsl-convention.md](querydsl-convention.md)
- [ddl-management.md](ddl-management.md)
