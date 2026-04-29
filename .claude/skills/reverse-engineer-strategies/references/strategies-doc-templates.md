# Strategies 문서 형식 가이드

각 레이어의 `strategies/README.md`와 컨벤션 문서의 형식을 정의한다.
분석 결과를 이 형식에 맞게 채워서 출력한다.

---

## strategies/README.md 공통 구조

모든 레이어의 `strategies/README.md`는 다음 세 섹션으로 구성된다.

레이어 이름이 플레이북과 다른 경우, 제목 아래에 **로컬 레이어명**과 **플레이북 개념 레이어 대응**을 함께 명시한다.
(예: `# api 계층 구현 전략 (플레이북: app 계층에 대응)`)

```markdown
# {Layer} 계층 구현 전략

[{layer}-layer-guidelines.md](../{layer}-layer-guidelines.md)의 보편 원칙(R1–RN) 위에서,
이 프로젝트가 선택한 {Layer} 계층 구현 전략을 정의한다.

---

## 이 프로젝트의 전략

**{전략 핵심}**: {구체적인 전략명}

**선택 이유**: {코드에서 발견한 근거 또는 추론된 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| {역할} | `{ComponentName}` | [{convention-file}.md]({convention-file}.md) |

**Post-Work Verification 체크리스트**:
- [{convention-file}.md]({convention-file}.md)
- ...

---

## 역할 정의

| 역할 | 설명 | 구현 예시 |
|-----|------|---------|
| {역할} | {설명} | `{예시}` |

---

## 전략 정의 템플릿

새 프로젝트에 적용할 때 "이 프로젝트의 전략" 섹션을 교체하기 위한 템플릿.

\`\`\`markdown
**{전략 핵심}**: {선택지}

**선택 이유**: {이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| {역할} | {컴포넌트명} | {링크 또는 `-`} |

**Post-Work Verification 체크리스트**:
- {컨벤션 문서 링크 목록}
\`\`\`
```

---

## 컨벤션 문서 공통 구조

모든 컨벤션 문서는 이 공통 구조를 기반으로 작성한다. **레이어 또는 컴포넌트에 특화된 섹션이 필요하면 추가할 수 있으나, 코드에서 관찰된 패턴이 없는 섹션은 생략한다.**

```markdown
# {Component} 컨벤션

> **핵심 규칙**: {한 줄 결론 — "A는 B이고 C를 절대 하지 않는다"}

---

## 네이밍 규칙

| 컴포넌트 | 패턴 | 예시 |
|---------|-----|-----|
| {컴포넌트명} | `{Pattern}` | `{Example}` |

---

## {주요 규칙 섹션 1}

> {이 규칙의 한 줄 결론}

{규칙 설명}

```kotlin
// 올바른 예
{code example}

// 잘못된 예  
{anti-pattern example}
```

---

## 컨벤션 문서 작성 지침

### 핵심 원칙

1. **사실 기반**: 코드에서 발견한 실제 패턴만 작성. 추측 금지.
2. **구체적인 예시**: 클래스명, 메서드 시그니처를 최대한 실제 코드에서 가져온다.
3. **이유 제시**: 각 규칙이 왜 그렇게 되어야 하는지 설명한다.
4. **발견된 것만 작성**: 코드에 없는 패턴은 컨벤션 문서에 포함하지 않는다.
   - 없는 항목은 `- {해당 없음}` 또는 섹션 자체를 생략한다.
5. **교과서 설명 배제**: 일반적인 아키텍처 원칙이나 패턴 이론은 쓰지 않는다. 이 프로젝트가 실제로 선택한 방식만 기술한다.
6. **관찰 결과 중심**: 실제 클래스명, 패키지 구조, 어노테이션, 구현 방식 같은 코드에서 직접 확인한 사실을 기술의 근거로 삼는다.
7. **근거 약한 내용 제외**: 코드에서 충분히 뒷받침되지 않는 내용은 "확인 필요" 표기도 하지 않고 문서에서 아예 제외한다.
8. **병합 모드의 제거 원칙**: 코드에서 더 이상 보이지 않는 패턴의 문서는 병합 모드에서도 제거 후보로 본다.

---

### 세부 컨벤션 문서를 만들 조건

- 코드베이스에서 반복적으로 사용되는 구현 패턴이 확인되면 그 패턴에 맞는 새 컨벤션 문서를 추가한다.
- 문서 생성 기준은 플레이북 지식 시스템에 존재하는 문서 목록이 아니라 **실제 코드에서 관찰된 구현 전략**이다.
  - 플레이북에 템플릿이 없더라도: 코드에서 반복 패턴이 보이면 → 새 문서 생성
  - 플레이북에 템플릿이 있더라도: 코드에서 해당 패턴이 없으면 → 문서 생성하지 않음

새 컨벤션 문서 생성 시:
- **파일 명명 규칙**: `{역할-키워드}-convention.md` (예: `scheduler-convention.md`, `batch-job-convention.md`, `cache-strategy-convention.md`)
- `strategies/README.md`의 역할별 컴포넌트 표와 Post-Work Verification 체크리스트에 해당 항목을 추가한다.

---

### 디폴트 파일 구성 예시

일반적인 Spring Boot 프로젝트에서 자주 등장하는 패턴의 예시다.
실제 생성 파일은 코드베이스 분석 결과에 따라 늘어나거나 줄어든다.

```
{출력경로}/
├── app/strategies/
│   ├── README.md
│   ├── api-convention.md            (Controller/DTO 패턴 발견 시)
│   ├── rest-design-convention.md    (REST 설계 규칙 발견 시)
│   ├── exception-handling-convention.md  (GlobalExceptionHandler 발견 시)
│   ├── file-structure.md            (패키지 구조가 명확할 때)
│   └── common.md                    (표현 계층 전역 관심사 — security/logging/config 등이 존재할 때)
├── application/strategies/
│   ├── README.md
│   ├── use-case-convention.md       (UseCase 패턴 발견 시)
│   ├── flow-convention.md           (Flow/Orchestrator 발견 시)
│   ├── validator-convention.md      (Validator 발견 시)
│   ├── handler-convention.md        (Handler/ACL 발견 시)
│   ├── policy-convention.md         (Policy/Strategy 발견 시)
│   ├── event-handler-convention.md  (EventHandler 발견 시)
│   └── mapper-convention.md         (Mapper/Assembler 발견 시)
├── domain/strategies/
│   ├── README.md
│   ├── domain-model-convention.md   (Entity/VO 패턴 발견 시)
│   └── exception-convention.md      (도메인 예외 패턴 발견 시)
├── storage/strategies/
│   ├── README.md
│   ├── storage-adapter-convention.md  (Adapter 패턴 발견 시)
│   └── {orm}-convention.md            (QueryDsl/JOOQ 등 발견 시)
└── external/strategies/
    ├── README.md
    ├── api-client-{http-client}.md    (HTTP 클라이언트 발견 시)
    └── api-client-logging.md          (로깅 패턴 발견 시)
```

