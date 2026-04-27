# App Module — File Structure

## 원칙

- 파일 구조는 관심사를 시각화한다. `common/`과 `{domain}/`의 분리는 도메인 횡단 관심사와 도메인별 책임을 위치로 드러낸다.
- 변환 로직은 DTO 옆에 있지 않다. Extension 파일이 `dto/` 밖에 있는 이유는 변환이 DTO의 책임이 아니라 경계 계층의 책임이기 때문이다.
- 도메인 패키지 간 직접 참조는 허용하지 않는다. 공유 로직은 반드시 `common/`을 경유한다.

---

## 개요

app 모듈의 최상위 디렉토리는 `common`과 `{domain}`으로 구성한다.

| 디렉토리 | 역할 |
|---------|------|
| `common/` | 표현 계층 전역 관심사. 특정 도메인에 의존하지 않는다 |
| `{domain}/` | 도메인별 표현 객체와 진입점 |

---

## 전체 구조 트리

```
:app:{module}/
├── common/
│   ├── config/          ← WebMvc, Jackson, CORS, Interceptor 등
│   ├── advice/          ← @RestControllerAdvice, 글로벌 예외 핸들러
│   ├── response/        ← BaseResponse<T>, ErrorResponse, PageResponse<T>
│   ├── security/        ← SecurityFilterChain, 인증 필터, 인가 규칙
│   ├── logging/         ← 요청·응답 로그 필터, MDC 설정, 마스킹
│   └── validator/       ← 커스텀 Bean Validation, 공통 포맷 Validator
└── {domain}/
    ├── {Entity}Controller.kt
    ├── {Domain}DtoExtension.kt   ← dto/ 밖에 위치
    └── dto/
        ├── {Domain}Requests.kt
        └── {Domain}Responses.kt  ← 필요 시만
```

각 공통 패키지의 역할·포함 대상·금지 사항 → [common 패키지 상세](file-structure/common.md)

---

## `{domain}/` 규칙

모든 도메인별 기능은 반드시 `{domain}/` 하위에 위치한다.

- `{domain}`은 단수형 소문자 (예: `auth`, `partner`, `catalog`, `inventory`, `stock`)
- Controller와 Extension은 `dto/` 밖 최상위에 평탄(flat) 배치한다.

### Controller

| 항목 | 규칙 |
|------|------|
| 위치 | `{domain}/{Entity}Controller.kt` |
| 역할 | 요청 수신 → UseCase 위임 → 응답 반환만 담당 |
| 금지 | 비즈니스 로직, 도메인 객체 직접 조작 |

### Extension

| 항목 | 규칙 |
|------|------|
| 위치 | `{domain}/{Domain}DtoExtension.kt` (`dto/` 바깥) |
| 역할 | Request → Command, Result → Response 변환 |
| 금지 | `dto/` 패키지 내부 배치 |

### DTO

| 항목 | 규칙 |
|------|------|
| Request 파일 | `dto/{Domain}Requests.kt` (복수). 1개면 단수형 |
| Response 파일 | `dto/{Domain}Responses.kt` (필요 시만) |
| 금지 | `toCommand()` 등 변환 로직을 DTO 내부에 포함 |

---

## 원칙

- `common`은 특정 도메인 클래스를 import하지 않는다.
- 도메인 패키지 간 직접 참조는 금지한다. 공통으로 사용하는 객체는 반드시 `common/`에 위치한다.
- app 모듈 간 코드 공유가 필요하면 `infra:internal` 또는 `core` 모듈을 경유한다.
