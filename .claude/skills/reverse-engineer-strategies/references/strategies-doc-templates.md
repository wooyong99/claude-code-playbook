# Strategies 문서 형식 가이드

각 레이어의 `strategies/README.md`와 컨벤션 문서의 형식을 정의한다.
분석 결과를 이 형식에 맞게 채워서 출력한다.

---

## strategies/README.md 공통 구조

모든 레이어의 `strategies/README.md`는 다음 세 섹션으로 구성된다.

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

## 레이어별 README.md 전략 섹션 예시

### Storage

```markdown
**ORM / 쿼리 빌더**: {JPA (Spring Data JPA) + QueryDsl | JOOQ | Exposed | 직접 JDBC}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| Port 구현체 | `{Entity}Adapter` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| 단순 CRUD / 단순 조회 | `{Entity}JpaRepository` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| 복잡 쿼리 / Projection | `{Entity}QueryDslRepository` | [querydsl-convention.md](querydsl-convention.md) |
| 도메인 ↔ Entity 변환 | `{Entity}Extension` | [storage-adapter-convention.md](storage-adapter-convention.md) |
```

### App

```markdown
**인증 방식**: {JWT Stateless | Session | OAuth2 Resource Server | API Key}
**멀티 테넌시**: {Header 기반 (`X-Tenant-Id`) | JWT claim 기반 | 없음}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 요청/응답 DTO | `{Domain}Requests`, `{Domain}Responses` | [api-convention.md](api-convention.md) |
| DTO ↔ Command/Result 변환 | `{Domain}DtoExtension` | [api-convention.md](api-convention.md) |
| REST 설계 규칙 | (규칙 문서) | [rest-design-convention.md](rest-design-convention.md) |
| 전역 예외 처리 | `GlobalExceptionHandler` | [exception-handling-convention.md](exception-handling-convention.md) |
```

### Application

```markdown
**오케스트레이션 패턴**: {UseCase + Flow 계층 분리 | UseCase + Service | 단일 Service}
**트랜잭션 소유**: {Flow | UseCase | Service}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 진입점 | `{Action}{Entity}UseCase` | [use-case-convention.md](use-case-convention.md) |
| 흐름 오케스트레이터 | `{Action}{Entity}Flow` | [flow-convention.md](flow-convention.md) |
| 검증기 | `{Entity}Validator` | [validator-convention.md](validator-convention.md) |
| ACL / 조율자 | `{Entity}Handler` | [handler-convention.md](handler-convention.md) |
| 전략 패턴 | `{Entity}Policy` | [policy-convention.md](policy-convention.md) |
| 이벤트 처리 | `{Entity}EventHandler` | [event-handler-convention.md](event-handler-convention.md) |
| 결과 변환 | `{Entity}Mapper` | [mapper-convention.md](mapper-convention.md) |
```

### Domain

```markdown
**도메인 모델 패턴**: {Static factory + Rich Domain Model | 일반 생성자 | data class only}
**예외 계층**: {CoreException 상속 | RuntimeException 직접 상속}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 도메인 엔티티 | `{Entity}` (id 기반 동등성) | [domain-model-convention.md](domain-model-convention.md) |
| 값 객체 | `{ValueObject}` (data class) | [domain-model-convention.md](domain-model-convention.md) |
| 도메인 예외 | `{Domain}ErrorCode`, `CoreException` | [exception-convention.md](exception-convention.md) |
```

### External

```markdown
**HTTP 클라이언트**: {WebClient (Reactive) | RestTemplate | Feign | OkHttp}
**추상화**: {Base ApiClient 클래스 | 개별 클라이언트}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| HTTP 설정 | `{Provider}Config` | [api-client-{http-client}.md](api-client-{http-client}.md) |
| API 클라이언트 | `{Provider}ApiClient` | [api-client-{http-client}.md](api-client-{http-client}.md) |
| 외부 어댑터 | `{Provider}Adapter` | [api-client-{http-client}.md](api-client-{http-client}.md) |
| 로깅 | (ExchangeFilterFunction 또는 Interceptor) | [api-client-logging.md](api-client-logging.md) |
```

---

## 컨벤션 문서 공통 구조

모든 컨벤션 문서는 동일한 섹션 순서를 따른다.

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

## 금지 패턴

- **{패턴명}**: {왜 금지인지}
- **{패턴명}**: {왜 금지인지}

---

## 체크리스트

- [ ] {검증 항목 1}
- [ ] {검증 항목 2}
```

---

## 컨벤션 문서 작성 지침

### 핵심 원칙

1. **사실 기반**: 코드에서 발견한 실제 패턴만 작성. 추측 금지.
2. **구체적인 예시**: 클래스명, 메서드 시그니처를 최대한 실제 코드에서 가져온다.
3. **이유 제시**: 각 규칙이 왜 그렇게 되어야 하는지 설명한다.
4. **발견된 것만 작성**: 코드에 없는 패턴은 컨벤션 문서에 포함하지 않는다.
   - 없는 항목은 `- {해당 없음}` 또는 섹션 자체를 생략한다.

### 플레이스홀더 사용

코드에서 특정 클래스명/값을 찾지 못한 경우 `{PlaceholderName}` 형식을 사용하되, 주석으로 "확인 필요" 표시를 남긴다.

```markdown
| Port 구현체 | `{Entity}Adapter` | <!-- 실제 클래스명 확인 필요 --> |
```

### 불확실한 정보 처리

코드를 분석했지만 확실하지 않은 부분은 문서 말미에 다음 형식으로 표기한다.

```markdown
---

## 확인 필요 사항

- **{항목}**: {불확실한 이유}. 사용자 확인 후 업데이트 필요.
```
