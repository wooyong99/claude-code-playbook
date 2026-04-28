# Storage 계층 구현 전략

[storage-layer-guidelines.md](../storage-layer-guidelines.md)의 보편 원칙(R1–R3) 위에서,
이 프로젝트가 선택한 Storage 계층 구현 전략을 정의한다.

새 프로젝트에 이 플레이북을 적용할 때는 **"이 프로젝트의 전략"** 섹션을 교체하고, 각 역할에 맞는 컨벤션 문서를 작성한다.

---

## 이 프로젝트의 전략

> **[커스터마이징 영역]** 새 프로젝트 적용 시 이 섹션을 아래 [템플릿](#전략-정의-템플릿)으로 교체한다.

**ORM / 쿼리 빌더**: JPA (Spring Data JPA) + QueryDsl

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| Port 구현체 | `{Entity}Adapter` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| 단순 CRUD / 단순 조회 | `{Entity}JpaRepository` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| 복잡 쿼리 / Projection | `{Entity}QueryDslRepository` | [querydsl-convention.md](querydsl-convention.md) |
| 도메인 ↔ Entity 변환 | `{Entity}Extension` | [storage-adapter-convention.md](storage-adapter-convention.md) |

**Post-Work Verification 체크리스트**:

- [storage-adapter-convention.md](storage-adapter-convention.md)
- [querydsl-convention.md](querydsl-convention.md)
- [../ddl-management.md](../ddl-management.md)

---

## 역할 정의

어떤 전략을 선택하든 아래 역할 중 필요한 것을 정의한다.

| 역할 | 설명 | 구현 예시 |
|-----|------|---------|
| Port 구현체 | application Port를 구현하고 저장소 컴포넌트에 위임 | `Adapter`, `Repository` |
| 단순 저장소 | 저장·삭제·단순 조회 | `JpaRepository`, `CrudRepository` |
| 복잡 쿼리 저장소 | 동적 조건·페이지네이션·Projection | `QueryDslRepository`, `JooqRepository` |
| 변환 컴포넌트 | 인프라 모델 ↔ 도메인 객체 변환 | `Extension`, `Mapper` |
| 인프라 모델 | DB 스키마 매핑 | `@Entity` class, JOOQ `Record` |

---

## 전략 정의 템플릿

```markdown
**ORM / 쿼리 빌더**: {JPA + QueryDsl / JOOQ / Exposed 중 선택}

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| Port 구현체 | {컴포넌트명} | {링크 또는 `-`} |
| 단순 저장소 | {컴포넌트명} | {링크 또는 `-`} |
| 복잡 쿼리 저장소 | {컴포넌트명} | {링크 또는 `-`} |
| 변환 컴포넌트 | {컴포넌트명} | {링크 또는 `-`} |

**Post-Work Verification 체크리스트**:
- {컨벤션 문서 링크 목록}
- [../ddl-management.md](../ddl-management.md)
```
