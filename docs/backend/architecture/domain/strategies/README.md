# Domain 계층 구현 전략

[domain-layer-guidelines.md](../domain-layer-guidelines.md)의 보편 원칙(R1–R4) 위에서,
이 프로젝트가 선택한 Domain 계층 내부 구현 전략을 정의한다.

---

## 이 프로젝트의 전략

> **[커스터마이징 영역]** 새 프로젝트 적용 시 이 섹션을 아래 템플릿으로 교체한다.

**전략명**: 정적 팩토리 + 행위 위임 (Rich Domain Model)

**흐름**: 외부 호출 → `companion object` 팩토리 메서드(`create` / `reconstitute`) → 도메인 객체 생성(불변식 검증) → 행위 메서드를 통한 상태 전이

**선택 이유**: 생성 경로를 팩토리 이름으로 구분해 `create`(신규)와 `reconstitute`(DB 복원) 불변식을 독립적으로 제어한다. `init` 재실행 문제를 원천 차단하고, 비즈니스 판단을 도메인 메서드에 캡슐화(Tell, Don't Ask)하여 정책 변경 시 호출부 수정을 최소화한다.

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 도메인 모델 (식별자 기반 동등성) | Entity | [domain-model-convention.md](domain-model-convention.md) |
| 도메인 모델 (값 기반 불변 객체) | Value Object | [domain-model-convention.md](domain-model-convention.md) |
| 비즈니스 규칙 위반 표현 | ErrorCode + CoreException | [exception-convention.md](exception-convention.md) |

**Post-Work Verification 체크리스트**:
- [domain-model-convention.md](domain-model-convention.md)
- [exception-convention.md](exception-convention.md)

---

## 역할 정의

| 역할 | 설명 | 구현 예시 |
|-----|------|---------|
| Entity | 식별자(id)로 동등성 판단. `private constructor` + 팩토리. 상태 전이 행위 메서드 보유 | `Order`, `User`, `Tenant` |
| Value Object | 값 자체로 동등성 판단하는 불변 객체(`data class`). 연산 메서드 보유 가능 | `Money`, `Address`, `Email` |
| ErrorCode | 도메인별 예외 코드 enum. `ErrorCode` 인터페이스 구현 | `OrderErrorCode`, `TenantErrorCode` |
| CoreException | 모든 도메인 예외의 기반 클래스. `ErrorCode`를 받아 단일 핸들러로 처리 | `CoreException(errorCode)` |

---

## 전략 정의 템플릿

```markdown
**전략명**: {전략 이름}

**흐름**: {컴포넌트 흐름 다이어그램}

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| {역할} | {컴포넌트명} | [{component}-convention.md]({component}-convention.md) |

**Post-Work Verification 체크리스트**:
- [{component}-convention.md]({component}-convention.md)
```
