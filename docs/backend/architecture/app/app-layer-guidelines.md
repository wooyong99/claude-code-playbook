# App Layer Guidelines

## App 계층의 본질적 책임

app 계층은 HTTP 요청을 수신하여 application 계층에 위임하고, 그 결과를 HTTP 응답으로 변환하는 경계 역할을 담당한다.

1. **HTTP 진입점 제공**: Controller가 Request DTO를 수신하고 UseCase를 통해 application 계층에 위임한다.
2. **HTTP 관심사 격리**: Spring·Jakarta 의존 코드(어노테이션, HTTP 상태 코드, 예외)를 app 계층 내에 가두고 application·domain 계층으로 누출시키지 않는다.
3. **예외 응답 단일화**: 모든 예외를 `GlobalExceptionHandler` 한 곳에서 HTTP 응답으로 변환한다.

```
클라이언트
  ↓ HTTP
app — Controller → Request DTO → Extension → Command
        ↓ UseCase 호출
  application
        ↓ Result
      Controller → Extension → BaseResponse 래핑
  ↓ HTTP
클라이언트

(예외) → GlobalExceptionHandler → BaseResponse.error 래핑 → HTTP 응답
```

---

## 반드시 지켜야 할 규칙

- **R1. HTTP 관심사 격리** — Controller·DTO·Extension은 HTTP 계층의 관심사만 처리한다. 비즈니스 로직은 application 계층에 위임한다.
- **R2. 예외 처리 중앙 집중화** — 모든 예외는 `GlobalExceptionHandler`에서 일괄 처리한다. Controller에 개별 `try-catch`를 두지 않는다.
- **R3. 도메인 모델 계층 경계** — Controller는 application 계층의 Command/Result와 소통하고, domain 모듈의 엔티티·값 객체를 직접 다루지 않는다.
- **R4. 변환 책임 분리** — Request DTO → Command, Result → Response 변환은 `{Domain}DtoExtension.kt`가 담당한다. DTO 자신이 변환 로직을 보유하지 않는다.

---

## 금지 규칙 / 안티패턴

- **Controller 내 비즈니스 로직** — UseCase 위임 없이 Controller에서 도메인 규칙을 검증·계산하면 app 계층과 application 계층의 경계가 무너진다.
- **Command 직접 바인딩** — application 계층 Command를 `@RequestBody`로 직접 수신하면 HTTP 계층과 application 계층의 계약이 결합된다.
- **분산 예외 처리** — Controller나 개별 Bean에 `@ExceptionHandler`·`try-catch`를 분산 배치하면 처리 일관성이 깨지고 핸들러 누락이 발생한다.
- **domain 모듈의 Spring 의존** — `HttpStatus`·Spring 어노테이션을 domain 모듈에서 참조하면 계층 독립성이 무너진다.
- **DTO 내 변환 로직** — `toCommand()` 등의 변환 메서드를 DTO 내부에 두면 HTTP 계층 교체 시 DTO를 함께 수정해야 한다.

---

## 이 프로젝트의 로컬 컨벤션

인증 방식·컨텍스트 전파·멀티테넌시 처리는 프로젝트 요구사항에 따라 달라진다. 어떤 전략을 선택하든 R1–R4는 반드시 지킨다. 역할 정의, 전략 선택 기준, 이 프로젝트의 선택 → [`strategies/`](strategies/README.md)

### 공통 규칙

**테스트**: Controller 테스트는 `@WebMvcTest` + MockMvc로 요청·응답 형태와 검증 어노테이션을 검증한다. UseCase는 mock으로 격리하고, `GlobalExceptionHandler` 핸들러별 응답 포맷·HTTP 상태 코드를 테스트한다.

**파일 구조**: app 모듈은 `common/`(표현 계층 전역 관심사)과 `{domain}/`(도메인별 표현 객체) 두 최상위 디렉토리로 구성한다. → [file-structure.md](strategies/file-structure.md) · [common.md](strategies/common.md)

### Post-Work Verification

구현 완료 후 생성·수정한 파일을 직접 읽어 아래 각 문서의 체크리스트를 대조한다.

이 프로젝트의 체크리스트 문서 목록 → [`strategies/`](strategies/README.md)
