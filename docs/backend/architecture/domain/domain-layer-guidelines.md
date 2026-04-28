# Domain Layer Guidelines

## Domain 계층의 본질적 책임

domain 계층은 순수 비즈니스 개념과 규칙을 표현하는 계층이다. 외부 프레임워크에 의존하지 않으므로 인프라 변경이 도메인 로직을 침범하지 않는다.

1. **비즈니스 개념 표현**: 도메인 언어로 비즈니스 개념과 관계를 코드로 구현한다.
2. **불변식 보호**: 도메인 객체가 스스로 생성과 상태 변경을 통제하여 잘못된 상태를 막는다.
3. **예외 계층 소유**: 비즈니스 규칙 위반을 도메인 소유 예외로 표현하며, HTTP 개념에 의존하지 않는다.

---

## 반드시 지켜야 할 규칙

- **R1. 순수성** — Spring, JPA, Jackson 등 외부 프레임워크에 의존하지 않는다. 순수 Kotlin / Java 표준 라이브러리만 사용한다.
- **R2. 불변식 보호** — 도메인 객체는 스스로 생성·상태 변경을 통제한다. 외부에서 도메인 규칙을 우회하는 경로를 허용하지 않는다.
- **R3. 도메인 경계** — 다른 도메인의 객체를 직접 포함하지 않는다. ID 참조로 경계를 유지한다.
- **R4. 예외 계층 소유** — 비즈니스 규칙 위반은 도메인 계층이 소유한 예외 계층(`ErrorCode`, `CoreException`)으로 표현한다. HTTP 프로토콜 개념(`HttpStatus`)에 의존하지 않는다.

---

## 금지 규칙 / 안티패턴

- **프레임워크 import** — `@Entity`, `@Component`, `@JsonProperty`, `HttpStatus` 등을 import하면 인프라 변경이 도메인 계층에 전파된다.
- **범용 ErrorCode enum** — 여러 도메인이 공유하는 `CommonErrorCode`는 예외의 소속을 모호하게 만들고 도메인 경계를 무너뜨린다.
- **HTTP 개념 직접 의존** — 도메인 계층이 `HttpStatus` 같은 프로토콜 타입을 직접 사용하면 표현 계층 변경이 도메인에 영향을 준다.

---

## 이 프로젝트의 로컬 컨벤션

내부 구성 방식은 프로젝트 환경·팀 선호에 따라 자유롭게 선택할 수 있다. 어떤 전략을 선택하든 R1–R4는 반드시 지킨다. 역할 정의, 전략 선택 기준, 이 프로젝트의 선택 → [`strategies/`](strategies/README.md)

### 파일 구조

```
{domain-module}/
└── src/main/kotlin/{your.package}/
    ├── {domain}/
    │   ├── {Entity}.kt              ← Entity 도메인 모델
    │   ├── {ValueObject}.kt         ← Value Object (data class)
    │   ├── {Entity}Status.kt        ← 상태 enum (필요 시)
    │   └── {Domain}ErrorCode.kt     ← 도메인별 ErrorCode enum
    │
    └── exception/
        ├── ErrorCode.kt             ← ErrorCode 인터페이스
        ├── CoreErrorType.kt         ← HTTP 의미론적 분류 (Spring 미의존)
        └── CoreException.kt         ← 기반 예외 클래스
```

### 공통 규칙

**테스팅**: 도메인 모델은 순수 Kotlin이므로 프레임워크 없이 단위 테스트한다.

**의존성 방향**:

```
domain (ErrorCode, CoreException, Entity, Value Object)
  ↑
application (Flow/UseCase에서 도메인 객체 조립 및 CoreException throw)
  ↑
app (GlobalExceptionHandler: CoreErrorType → HttpStatus 매핑)
```

### Post-Work Verification

구현 완료 후 생성·수정한 파일을 직접 읽어 아래 각 문서의 체크리스트를 대조한다.

이 프로젝트의 체크리스트 문서 목록 → [`strategies/README.md`](strategies/README.md)
