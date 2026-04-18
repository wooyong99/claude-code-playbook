# Storage Module Guidelines

`:infra:storage` 모듈의 구성 요소와 역할을 정의하는 **인덱스 문서**. 각 구성 요소의 상세 규칙·예시·판단 기준은 하위 컨벤션 문서를 따른다.

---

## Purpose

- 인프라 계층(`:infra:storage`)의 Port 구현체(Adapter)와 JPA 영속화 담당
- JPA Entity는 인프라 계층 밖으로 노출하지 않고, 도메인 객체로 변환하여 반환

---

## 레이어 개요

```
application (Port 인터페이스)
  ↑ 구현
storage  — Adapter → JpaRepository / QueryDslRepository → DB
             ↕ (변환)
          Entity ↔ Domain (Extension 함수)
```

- **Adapter**: application Port 인터페이스 구현체. JpaRepository / QueryDslRepository에 위임
- **JpaRepository**: Spring Data JPA. CRUD / 단순 조건 조회 담당
- **QueryDslRepository**: QueryDsl. 동적 조건 / 페이지네이션 / Projection / 복잡 조인 담당
- **Entity**: JPA 매핑 전용 클래스. 비즈니스 로직 포함 금지
- **Extension**: Entity ↔ Domain 변환 (`toDomain()` / `toEntity()`) 전용 파일

---

## 구성 요소별 상세 문서

| 구성 요소 | 핵심 규칙 (한 줄 요약) | 상세 문서 |
|-----------|---------------------|----------|
| Adapter / JpaRepository / Entity / Extension | `{Entity}Adapter` Port 구현 + Jpa(단순) · QueryDsl(복잡) 분리 + Entity는 JPA 매핑만 + Extension이 도메인 변환 | [storage-adapter-convention.md](storage-adapter-convention.md) |
| QueryDsl | `BooleanExpression?` 동적 조건, 2-step 조인 페이지네이션, `Projections.constructor` 기반 Projection | [querydsl-convention.md](querydsl-convention.md) |
| DDL 관리 | DDL 파일 위치 `sql/{domain}/{table}.sql`. 엔티티 신규/변경 시 **같은 커밋에서 DDL 동기화**. 버전 접두사 미사용 | [ddl-management.md](ddl-management.md) |

---

## File Structure

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

`StorageConfig`의 `basePackages`는 루트 패키지로 고정되어 있어, 하위 패키지를 추가해도 별도 설정 변경이 필요하지 않다.

---

## Testing

- Adapter / Repository 테스트는 `@DataJpaTest` 또는 `@SpringBootTest`로 실제 DB 연동 테스트한다.
- QueryDsl 동적 조건과 페이지네이션(2-step 조인)은 통합 테스트로 검증한다.
- Entity ↔ Domain 변환(`toDomain` / `toEntity`) 정합성을 테스트한다.

---

## Post-Work Verification

**가이드라인 문서를 참고했더라도 실제 생성된 코드에 반영되지 않은 부분이 있을 수 있다.** 구현 완료 후, 생성하거나 수정한 파일을 직접 읽어서 아래 각 하위 문서의 체크리스트를 하나씩 대조한다. 위반이 발견되면 즉시 수정하고 다시 검증한다.

각 문서 하단의 "체크리스트" 섹션을 참고한다.

- [storage-adapter-convention.md](storage-adapter-convention.md)
- [querydsl-convention.md](querydsl-convention.md)
- [ddl-management.md](ddl-management.md)
