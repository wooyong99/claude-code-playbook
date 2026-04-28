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

| 역할 | 컴포넌트 | 위치 | 컨벤션 문서 |
|-----|---------|------|-----------|
| Controller | `{Domain}Controller` | app 모듈 `{domain}/controller/` | [api-convention.md](api-convention.md) |
| 요청/응답 DTO | `{Domain}Requests`, `{Domain}Responses` | {app 모듈 로컬 또는 application 모듈 `inbound/`} | [api-convention.md](api-convention.md) |
| DTO ↔ Command 변환 | `{Domain}DtoExtension` | {app 모듈 또는 application 모듈} | [api-convention.md](api-convention.md) |
| REST 설계 규칙 | (규칙 문서) | - | [rest-design-convention.md](rest-design-convention.md) |
| 전역 예외 처리 | `GlobalExceptionHandler` | {app 모듈 또는 application 모듈} | [exception-handling-convention.md](exception-handling-convention.md) |
```

> **작성 전 코드에서 확인할 사항 (가정 금지)**:
> - Controller 파일의 실제 경로(`{domain}/controller/` 서브패키지인가 flat인가)
> - Request/Response DTO가 `{app_module}`에 로컬로 있는가, `{application_module}` inbound 패키지에서 직접 가져오는가
> - DTO ↔ Command 변환 클래스가 `{app_module}`에 있는가, `{application_module}`에 있는가
> - 전역 예외 처리 클래스·공통 응답 클래스가 `{app_module}`에 있는가, `{application_module}`에 있는가
> - Security 설정 클래스가 `{app_module}`에 있는가, `{infra_module}`에 있는가
>
> 공통 관심사(예외처리, Security)가 **다른 모듈에 있는 경우**, 컴포넌트 표의 "위치" 칸에 실제 모듈명을 명시하고 해당 모듈의 strategies 문서에서 다룬다고 안내한다.

### Application

```markdown
**오케스트레이션 패턴**: {UseCase 인터페이스 + Service 구현체 | UseCase + Flow 계층 분리 | 단일 Service}
**트랜잭션 소유**: {Service | Flow | UseCase}
**CQRS 분리 수준**: {패키지 + 인터페이스 수준 분리 | 인터페이스만 분리 | 미분리}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 진입점 (Inbound Port) | `{Domain}UseCase` (interface) | [use-case-convention.md](use-case-convention.md) |
| 쓰기 구현체 | `{Domain}Service` (@Service) | [use-case-convention.md](use-case-convention.md) |
| 읽기 구현체 | `Query{Domain}Service` (@Service) | [use-case-convention.md](use-case-convention.md) |
| ACL / 크로스도메인 조율자 | `{Domain}Handler` (@Component) | [handler-convention.md](handler-convention.md) |
| 이벤트 처리 | `{Domain}EventHandler` (@Component) | [event-handler-convention.md](event-handler-convention.md) |
| Outbound Port | `{Domain}Port` (interface) | [use-case-convention.md](use-case-convention.md) |
```

> 코드베이스에 없는 역할의 행과 해당 컨벤션 문서 링크는 제거한다.

### Infrastructure

Infrastructure 레이어는 JPA 영속성 어댑터(storage 역할)와 Spring Security/Config가 **같은 모듈**에 공존할 때 사용한다.
별도의 `storage` 모듈이 있을 경우 해당 모듈은 Storage 템플릿을 사용한다.

```markdown
**ORM / 쿼리 빌더**: {JPA (Spring Data JPA) + QueryDsl | JOOQ | Exposed | 직접 JDBC}
**인증 방식**: {JWT Stateless | Session | OAuth2 Resource Server | API Key}
**멀티 테넌시**: {JWT claim 기반 | Header 기반 (`X-Tenant-Id`) | 없음}

**선택 이유**: {코드에서 발견한 근거}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| Port 구현체 | `{Entity}Adapter` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| 단순 CRUD / 단순 조회 | `{Entity}JpaRepository` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| 복잡 쿼리 / Projection | `{Entity}QueryDslRepository` | [querydsl-convention.md](querydsl-convention.md) |
| 도메인 ↔ Entity 변환 | `{Entity}Extension` | [storage-adapter-convention.md](storage-adapter-convention.md) |
| Security 설정 | `SecurityConfig` | [security-convention.md](security-convention.md) |
| JWT 인증 필터 | `JwtAuthenticationFilter` | [security-convention.md](security-convention.md) |
```

> 특정 역할의 클래스가 이 모듈이 아닌 다른 모듈에 있으면 해당 행을 제거하고 실제 모듈의 strategies 문서에서 다룬다.

---

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
