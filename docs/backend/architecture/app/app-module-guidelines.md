# App Module Guidelines

`:app:storefront`, `:app:backoffice`, `:app:admin` 등 **모든 app 모듈**에 공통으로 적용되는 규칙을 정의하는 **인덱스 문서**. 각 구성 요소의 상세 규칙·예시·판단 기준은 하위 컨벤션 문서를 따른다.

---

## Purpose

- REST API 진입점 (Controller, Request/Response DTO)
- HTTP / Spring 관심사를 application 계층으로 누출시키지 않는 경계 역할
- 도메인 예외 → HTTP 응답 변환의 단일 책임 지점 (`GlobalExceptionHandler`)

---

## 레이어 개요

```
클라이언트
  ↓ HTTP
app — Controller → Request DTO → Extension → Command
        ↓ (UseCase 호출)
  application
        ↓ (Result)
      Controller → (Extension) → BaseResponse 래핑
  ↓ HTTP
클라이언트

(예외) → GlobalExceptionHandler → BaseResponse.error 래핑 → HTTP 응답
```

- **Controller**: Request 수신 → UseCase 위임 → 응답 반환 (비즈니스 로직 금지)
- **Request / Response DTO**: HTTP 요청·응답 스키마 (Spring 검증 어노테이션만)
- **Extension**: Request → Command, Result → Response 변환 (`{Domain}DtoExtension.kt`)
- **GlobalExceptionHandler**: 모든 예외의 단일 처리 지점
- **BaseErrorCode / BaseError / BaseResponse**: HTTP 응답 표준 포맷
- **ErrorTypeExtension**: `CoreErrorType → HttpStatus` 매핑

---

## 구성 요소별 상세 문서

| 구성 요소 | 핵심 규칙 (한 줄 요약) | 상세 문서 |
|-----------|---------------------|----------|
| Controller / Request / Response / Extension | Controller는 UseCase 위임만. DTO는 검증 어노테이션만. 변환은 Extension. `BaseResponse<T>` 응답 | [api-convention.md](api-convention.md) |
| REST API 설계 (URL / Plural / ID 위치 / Pagination) | 복수형 + kebab-case + `/api/v{N}/`. 식별자는 Path, 필터·정렬·페이지네이션은 Query. 커서 기반 우선, 필요 시 오프셋 | [rest-design-convention.md](rest-design-convention.md) |
| Exception Handling | 모든 예외는 `GlobalExceptionHandler`에서 일괄 처리. `CoreErrorType → HttpStatus` 매핑은 `ErrorTypeExtension` 단독 책임 | [exception-handling-convention.md](exception-handling-convention.md) |

---

## File Structure

app 모듈의 최상위 디렉토리는 `common/`과 `{domain}/`으로 구성한다.

```
:app:{module}/
├── common/
│   ├── config/      ← WebMvc, Jackson, CORS, Interceptor 등
│   ├── advice/      ← @RestControllerAdvice, GlobalExceptionHandler
│   ├── response/    ← BaseResponse<T>, ErrorResponse, PageResponse<T>
│   ├── security/    ← SecurityFilterChain, 인증 필터, 인가 규칙
│   ├── logging/     ← 요청·응답 로그 필터, MDC 설정, 마스킹
│   └── validator/   ← 커스텀 Bean Validation, 공통 포맷 Validator
└── {domain}/
    ├── {Entity}Controller.kt        ← Controller (flat 배치)
    ├── {Domain}DtoExtension.kt      ← 매핑 Extension (dto/ 밖)
    └── dto/
        ├── {Domain}Requests.kt      ← Request DTO 모음
        └── {Domain}Responses.kt     ← Response DTO 모음 (필요 시만)
```

상세 규칙 → [file-structure.md](file-structure.md) · [common/ 패키지 상세](file-structure/common.md)

---

## Testing

- Controller 테스트는 `@WebMvcTest` + MockMvc로 요청/응답 형태와 검증 어노테이션을 테스트한다.
- Request DTO의 Spring 검증 어노테이션이 올바르게 동작하는지 테스트한다.
- UseCase는 mock으로 격리하고, Controller는 요청 파싱과 응답 포맷에만 집중한다.
- `GlobalExceptionHandler` 핸들러별로 응답 포맷과 HTTP 상태 코드를 테스트한다.

---

## Post-Work Verification

**가이드라인 문서를 참고했더라도 실제 생성된 코드에 반영되지 않은 부분이 있을 수 있다.** 구현 완료 후, 생성하거나 수정한 파일을 직접 읽어서 아래 각 하위 문서의 체크리스트를 하나씩 대조한다. 위반이 발견되면 즉시 수정하고 다시 검증한다.

각 문서 하단의 "체크리스트" 섹션을 참고한다.

- [api-convention.md](api-convention.md)
- [rest-design-convention.md](rest-design-convention.md)
- [exception-handling-convention.md](exception-handling-convention.md)
