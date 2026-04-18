# DDL Management

엔티티 스키마와 DDL 파일을 관리하는 규칙.

---

## 파일 위치

- DDL: `sql/{domain}/{table}.sql`
- 초기 seed 데이터: `sql/{domain}/seed.sql`

---

## 버전 관리 방식

- Flyway/Liquibase 스타일의 버전 접두사(`V1__`, `V2__`)를 **사용하지 않는다**.
- 스키마 변경 시 해당 `.sql` 파일을 직접 수정한다.
- 엔티티 신규/변경 시 대응하는 DDL 파일도 **같은 커밋에서 함께 갱신**한다 (항상 동기화).
