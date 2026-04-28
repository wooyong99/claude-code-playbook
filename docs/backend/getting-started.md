# Getting Started

백엔드 로컬 개발 환경 구성 및 실행 안내.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| 언어 | Kotlin 1.9.25 |
| 프레임워크 | Spring Boot 3.5.11 |
| 데이터 접근 | Spring Data JPA / Hibernate |
| 데이터베이스 | MySQL 8.0 (운영), H2 (테스트) |
| 빌드 | Gradle 8.x (Kotlin DSL) |
| 테스트 | JUnit 5, Spring Boot Test |
| 런타임 | Java 17 |

---

## 실행

```bash
docker-compose -f docker-compose-local.yml up -d
./gradlew build -x test --parallel
./gradlew :{app-module}:bootRun --args='--spring.profiles.active=local'
```

---

## 프로필

| 프로필 | 용도 |
|--------|------|
| `local` | 로컬 개발 (LocalFileStorage) |
| `storage` | MySQL DB 설정 (항상 포함) |
