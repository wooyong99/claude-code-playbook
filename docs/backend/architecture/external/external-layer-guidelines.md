# External Layer Guidelines

## External 계층의 본질적 책임

외부 시스템(PG, 상품권 서비스, 알림 서비스 등)과의 통신을 담당하는 Outbound Adapter 계층이다. application 계층이 정의한 Outbound Port를 구현하여 외부 시스템 호출을 캡슐화하고, 외부 서비스의 오류·응답을 내부 표현으로 변환한다.

```
application (Outbound Port 정의)
        ↑ 구현
external  (Adapter → ApiClient → 외부 시스템)
```

1. **Port 구현**: application이 선언한 Outbound Port의 구현체로서 외부 API 호출을 담당한다.
2. **경계 번역**: 외부 예외·응답 스키마를 Port Result 타입으로 변환해 상위 계층을 외부 기술 세부로부터 보호한다.
3. **격리 구성**: Mock Adapter를 통해 로컬 개발 환경을 실 외부 시스템과 분리한다.

---

## 반드시 지켜야 할 규칙

- **R1. 외부 변화는 내부로 전파하지 않는다** — Adapter가 Port 계약과 외부 스키마 사이를 분리하여, 외부 API 변경이 application 계층에 영향을 미치지 않게 한다.
- **R2. 외부 예외는 내부 표현으로 번역한다** — HTTP/네트워크 예외는 Provider 전용 sealed 예외 계층을 거쳐 Port Result로 변환하고, application 계층에 기술 예외가 노출되지 않는다.
- **R3. 의존성은 external → application → domain 방향으로만 흐른다** — Adapter는 Port 입·출력 타입 외의 도메인 객체를 직접 참조하지 않는다.
- **R4. Provider 단위로 패키지를 분리한다** — 한 패키지에 해당 Provider의 모든 구성 요소가 공존하며, Provider 간 코드가 혼재하지 않는다.

---

## 금지 규칙 / 안티패턴

- **HTTP/네트워크 예외를 Adapter 밖으로 전파** — application 계층이 Spring 프레임워크 세부에 결합된다.
- **여러 Provider가 같은 패키지를 공유** — 변경 영향도 파악이 어려워지고 단일 책임이 깨진다.
- **Adapter에서 Port 외의 도메인 객체 참조** — external이 domain을 직접 의존해 계층 의존 방향이 역전된다.
- **외부 DTO를 Port 시그니처로 노출** — 외부 스키마 변경이 application 계층까지 전파된다.

---

## 이 프로젝트의 로컬 컨벤션

### 공통 규칙

**패키지 구조**: Provider 단위 패키지 분리. 한 패키지에 Adapter·ApiClient·TokenHolder·Dto·ErrorCode·Exception·Config·Properties·Mock Adapter가 공존한다.

```
{external-module}/src/main/kotlin/{your.package}/
└── {provider}/
    ├── {Provider}{Function}Adapter.kt            ← Outbound Port 구현 (@Profile("!local"))
    ├── Mock{Function}Adapter.kt                  ← 로컬용 Mock 구현 (@Profile("local"))
    ├── {Provider}ApiClient.kt                    ← HTTP 호출
    ├── {Provider}TokenHolder.kt                  ← Bearer Token 캐싱 (필요한 경우)
    ├── {Provider}Dtos.kt                         ← 요청/응답 DTO
    ├── {Provider}ErrorCode.kt                    ← 외부 API 에러코드 → Port ErrorCode 번역
    ├── {Provider}Exception.kt                    ← sealed 예외 계층
    ├── {Provider}Config.kt                       ← HTTP 클라이언트 빈
    ├── {Provider}Properties.kt                   ← @ConfigurationProperties
    └── code/                                     ← Provider 전용 코드값 enum (필요한 경우)
        └── {Provider}{Category}Code.kt
```

복잡한 Provider는 `config/`, `deserializer/` 등 하위 패키지로 분리할 수 있으나, **한 Provider가 두 개 이상의 최상위 패키지를 점유하지 않는다**.

**구성 요소별 컨벤션 문서**:

| 구성 요소 | 핵심 규칙 (한 줄 요약) | 컨벤션 문서 |
|-----------|---------------------|-----------|
| Adapter | Outbound Port만 구현하고, 외부 예외는 Port Result로 번역한다 | [adapter-convention.md](adapter-convention.md) |
| ApiClient | HTTP 호출을 캡슐화하고 HTTP/네트워크 예외를 Provider 예외로 변환한다 | [api-client-convention.md](api-client-convention.md) |
| Dto | `@JsonIgnoreProperties(ignoreUnknown = true)` + `@JsonProperty`로 외부 스키마를 고정한다 | [dto-convention.md](dto-convention.md) |
| Exception | Provider별 `sealed class`로 API / Auth / Server / Network / Parsing 5종을 정의한다 | [exception-convention.md](exception-convention.md) |
| ErrorCode | `{Provider}ErrorCode` enum으로 외부 에러코드를 정의하고 `toPortErrorCode()`로 번역한다 | [errorcode-convention.md](errorcode-convention.md) |
| Config / Properties | HTTP 클라이언트는 Provider 전용 `@Qualifier` 빈으로, 설정값은 `@ConfigurationProperties`로 바인딩한다 | [config-convention.md](config-convention.md) |
| Mock Adapter | `@Profile("local")`로 실 Adapter(`@Profile("!local")`)와 대칭 구성한다 | [mock-adapter-convention.md](mock-adapter-convention.md) |

**ApiClient 구현 전략**: 사용 HTTP 클라이언트·로깅 방식은 프로젝트별로 다르다. 이 프로젝트의 선택 → [`strategies/`](strategies/README.md)

**Profile 관리**: 실 Adapter는 `@Profile("!local")`, Mock Adapter는 `@Profile("local")`로 선언해 동일 Port의 빈 충돌을 방지한다. 상세는 [mock-adapter-convention.md](mock-adapter-convention.md) "Profile 분기" 섹션 참고.

**테스트**: ApiClient 순수 로직(해시 계산, 직렬화 등)은 단위 테스트로 검증한다. Adapter는 ApiClient를 MockK로 대체한 BDD 테스트(`BehaviorSpec`)를 작성한다.

### Post-Work Verification

구현 완료 후 생성·수정한 파일을 직접 읽어 아래 각 문서의 체크리스트를 대조한다.

이 프로젝트의 체크리스트 문서 목록 → [`strategies/`](strategies/README.md)
