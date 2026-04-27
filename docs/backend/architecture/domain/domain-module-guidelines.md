# Domain Module Guidelines

## 원칙

- 도메인 계층은 순수하다. Spring, JPA, Jackson 같은 프레임워크 의존이 없어야 비즈니스 규칙이 인프라 변경에 영향받지 않는다.
- 도메인은 비즈니스 언어로 말한다. 외부에서 enum을 꺼내 판단하는 대신, 도메인 객체에게 질문하고 답을 얻는다 (Tell, Don't Ask).
- 불변식은 도메인이 보호한다. `private constructor` + 팩토리 메서드로 생성을 제어하고, 행위 메서드로만 상태를 변경한다.

---

`:core:domain` 모듈의 구성 요소와 역할을 정의하는 **인덱스 문서**. 각 구성 요소의 상세 규칙·예시·판단 기준은 하위 컨벤션 문서를 따른다.

---

## Purpose

- 순수 비즈니스 개념과 규칙을 표현하는 도메인 모델 계층
- 외부 프레임워크 의존 없이 Kotlin / Java 표준 라이브러리만 사용

---

## 레이어 개요

```
domain (ErrorCode, CoreException, Entity, Value Object)
  ↑
application (Flow/UseCase에서 도메인 객체 조립 및 CoreException throw)
  ↑
app (GlobalExceptionHandler: CoreErrorType → HttpStatus 매핑)
```

- **Entity**: 식별자(id)로 동등성을 판단하는 비즈니스 개념. 상태 전이 메서드 보유
- **Value Object**: 값 자체로 동등성을 판단하는 불변 객체 (`data class`)
- **ErrorCode**: 도메인별 enum으로 에러 코드를 상수화. `ErrorCode` 인터페이스 구현
- **CoreException**: 모든 도메인 예외의 기반 클래스. `ErrorCode`를 받아 통일된 핸들러로 처리

---

## 구성 요소별 상세 문서

| 구성 요소 | 핵심 규칙 (한 줄 요약) | 상세 문서 |
|-----------|---------------------|----------|
| Entity / Value Object | `private constructor` + `companion object` 팩토리. `init` 금지. Spring / JPA / Jackson 의존 금지. 상태 변경은 행위 메서드로만 | [domain-model-convention.md](domain-model-convention.md) |
| Exception / ErrorCode | 도메인별 `{Domain}ErrorCode` enum + `CoreException`. HTTP 매핑은 app 계층에서. 프레임워크(`HttpStatus`) 의존 금지 | [exception-convention.md](exception-convention.md) |

---

## File Structure

```
:core:domain/
└── src/main/kotlin/com/wooyong/demo/core/domain/
    ├── {domain}/
    │   ├── {Entity}.kt              ← Entity 도메인 모델
    │   ├── {ValueObject}.kt         ← Value Object (data class)
    │   ├── {Entity}Status.kt        ← 상태 enum (필요 시)
    │   └── {Domain}ErrorCode.kt     ← 도메인별 ErrorCode enum
    │
    └── exception/
        ├── ErrorCode.kt             ← ErrorCode 인터페이스
        ├── CoreErrorType.kt         ← HTTP 의미론적 분류 (Spring 미의존)
        └── CoreException.kt         ← 기본 예외 클래스
```

---

## Testing

- 도메인 모델은 순수 Kotlin이므로 프레임워크 없이 단위 테스트한다.
- 팩토리 메서드(`create` / `reconstitute`) 생성 경로를 테스트한다.
- 행위 메서드의 상태 전이와 전제조건(`require` / `check`) 위반을 테스트한다.
- Tell, Don't Ask 메서드(`isActive()`, `canCancel()` 등)의 판단 로직을 테스트한다.

---

## Post-Work Verification

**가이드라인 문서를 참고했더라도 실제 생성된 코드에 반영되지 않은 부분이 있을 수 있다.** 구현 완료 후, 생성하거나 수정한 파일을 직접 읽어서 아래 각 하위 문서의 체크리스트를 하나씩 대조한다. 위반이 발견되면 즉시 수정하고 다시 검증한다.

각 문서 하단의 "체크리스트" 섹션을 참고한다.

- [domain-model-convention.md](domain-model-convention.md)
- [exception-convention.md](exception-convention.md)
