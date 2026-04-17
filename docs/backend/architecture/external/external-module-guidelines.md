# External Module Guidelines

`:infra:external` 모듈의 구성 요소와 역할을 정의하는 **인덱스 문서**. 각 구성 요소의 상세 규칙·예시·판단 기준은 하위 컨벤션 문서를 따른다.

---

## Purpose

외부 시스템(PG, 상품권 서비스, 알림톡, 메일 등)과의 통신을 담당하는 **Outbound Adapter 계층**. `application` 모듈이 정의한 Outbound Port를 구현하여 외부 시스템 호출을 캡슐화하고, 외부 서비스의 오류·응답을 내부 도메인 표현으로 변환한다.

---

## 레이어 개요

```
application (Outbound Port 정의)
        ↑ 구현
external  (Adapter → ApiClient → 외부 시스템)
```

- **Adapter**: Outbound Port 구현체. Port 요청/결과 타입과 외부 DTO를 매핑하고, 외부 예외를 Port Result로 번역한다.
- **ApiClient**: `RestClient` 기반 HTTP 호출 전담. `@ExternalApiLogging`을 부착하고 HTTP/네트워크 예외를 Provider 전용 예외로 변환한다.
- **Dto**: 외부 API 요청/응답 전용 데이터 클래스. Provider의 JSON 스키마를 1:1로 표현한다.
- **Exception**: Provider 전용 sealed 예외 계층. Adapter가 Result로 변환할 때 분기 근거가 된다.
- **Config / Properties**: `RestClient` 빈과 `@ConfigurationProperties` 바인딩.
- **Mock Adapter**: 로컬 개발용 Port 구현체. Profile 기반으로 실 Adapter와 대칭 구성한다.

모듈 의존성은 `external → application → domain` 방향으로만 흐른다. Adapter는 Port가 정의한 입·출력 타입 외의 도메인 객체를 참조하지 않는다.

---

## 구성 요소별 상세 문서

| 구성 요소 | 핵심 규칙 (한 줄 요약) | 상세 문서 |
|-----------|---------------------|----------|
| Adapter | Outbound Port만 구현하고, 외부 예외는 Port Result로 번역한다 | [adapter-convention.md](adapter-convention.md) |
| ApiClient | `RestClient` 호출을 캡슐화하고 HTTP/네트워크 예외를 Provider 예외로 변환한다 | [api-client-convention.md](api-client-convention.md) |
| Dto | `@JsonIgnoreProperties(ignoreUnknown = true)` + `@JsonProperty`로 외부 스키마를 고정한다 | [dto-convention.md](dto-convention.md) |
| Exception | Provider별 `sealed class`로 분류하고 API / HTTP / Network / Parsing 4종을 기본으로 둔다 | [exception-convention.md](exception-convention.md) |
| Config / Properties | `RestClient`는 Provider 전용 `@Qualifier` 빈으로, 설정값은 `@ConfigurationProperties`로 바인딩한다 | [config-convention.md](config-convention.md) |
| Mock Adapter | `@Profile("local")`로 실 Adapter(`@Profile("!local")`)와 대칭 구성한다 | [mock-adapter-convention.md](mock-adapter-convention.md) |

---

## 모듈 공통 규칙

### 패키지 구조

Provider 단위로 패키지를 분리한다. 한 패키지에 해당 Provider의 Adapter·ApiClient·Dto·Exception·Config·Properties·Mock Adapter가 공존한다.

```
backend/infra/external/src/main/kotlin/com/wooyong/musinsa/
└── {provider}/
    ├── {Provider}{Function}Adapter.kt            ← Outbound Port 구현
    ├── {Provider}{Function}MockAdapter.kt        ← 로컬용 Mock 구현
    ├── {Provider}ServiceApiClient.kt             ← HTTP 호출
    ├── {Provider}ServiceDto.kt                   ← 요청/응답 DTO
    ├── {Provider}ServiceException.kt             ← sealed 예외 계층
    ├── {Provider}ServiceConfig.kt                ← RestClient 빈
    └── {Provider}ServiceProperties.kt            ← @ConfigurationProperties
```

복잡한 Provider(예: `keyin`)는 `config/`, `deserializer/`, `pg/` 등 하위 패키지로 분리할 수 있으나, **한 Provider가 두 개 이상의 최상위 패키지를 점유하지 않는다**.

### Profile 관리

- 외부 호출이 실 Provider에 의존하는 Adapter는 `@Profile("!local")`로 제한하고, 로컬 개발용 Mock Adapter는 `@Profile("local")`을 부여한다.
- 실 Adapter와 Mock Adapter가 동일 Port를 구현하므로, 프로파일 분기로 빈 충돌을 방지한다.

> 상세는 [mock-adapter-convention.md](mock-adapter-convention.md) "Profile 분기" 섹션 참고

### 로깅

- API 호출 로그는 반드시 `@ExternalApiLogging`으로 남긴다. AOP가 요청/응답/소요시간을 일관된 포맷으로 기록한다.
- Adapter 수준의 비즈니스 맥락 로그(요청 식별자, 결과 상태)는 `logInfo` / `logWarn`을 사용한다. 민감정보(핀 번호, 카드번호 등)는 마스킹한다(`take(4) + "****"`).

> API 호출 로그 규칙은 [api-client-convention.md](api-client-convention.md) "로깅" 섹션 참고

### 테스트

- ApiClient의 순수 로직(해시 계산, 직렬화 등)은 단위 테스트로 검증한다. 예: `PayletterExtensionTest`.
- Adapter는 ApiClient를 MockK로 대체한 BDD 테스트(`BehaviorSpec`)를 작성한다.
- Mock Adapter는 테스트 시나리오 분기 규칙(`pinNo.contains("INVALID")` 등)이 있으므로, 해당 분기가 유효한지 단위 테스트로 검증한다.

---

## Post-Work Verification

구현 완료 후 각 하위 문서의 "체크리스트" 섹션을 대조한다.

- [adapter-convention.md](adapter-convention.md)
- [api-client-convention.md](api-client-convention.md)
- [dto-convention.md](dto-convention.md)
- [exception-convention.md](exception-convention.md)
- [config-convention.md](config-convention.md)
- [mock-adapter-convention.md](mock-adapter-convention.md)
