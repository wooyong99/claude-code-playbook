# Application Module Guidelines

`:core:application` 모듈의 구성 요소와 역할을 정의하는 **인덱스 문서**. 각 구성 요소의 상세 규칙·예시·판단 기준은 하위 컨벤션 문서를 따른다.

---

## Purpose

- 유스케이스/서비스 계층
- 포트/어댑터 경계를 유지하며 도메인을 조립

---

## 레이어 개요

```
UseCase → Flow → Validator / Handler / Policy → Port (interface)
```

- **UseCase**: inbound entry point. Flow 조합만 담당
- **Flow**: 하나의 업무 흐름 실행. Validator/Handler/Policy/Port 조합
- **Validator**: 조회된 객체를 받아 규칙만 검증 (Port 주입 금지)
- **Handler**: 여러 도메인에서 재사용되는 공통 로직 / Flow의 ACL
- **Policy**: 정책에 따른 행위 분기 (`List<Policy>` + `supports(type)` 디스패치)
- **EventHandler**: 트랜잭션 커밋 이후 부수 효과 처리
- **Mapper**: 도메인 객체 → Result DTO 변환
- **Port**: outbound 인터페이스 (Repository, 외부 서비스 추상)

---

## 구성 요소별 상세 문서

| 구성 요소 | 핵심 규칙 (한 줄 요약) | 상세 문서 |
|-----------|---------------------|----------|
| UseCase | `@Service` 선언. Flow 조합만 담당. `@Transactional`은 원칙적으로 Flow에 선언 | [use-case-convention.md](use-case-convention.md) |
| Flow | `@Component` 선언. 쓰기 `@Transactional`, 읽기 `@Transactional(readOnly = true)`. Flow → Flow 직접 호출 금지 | [flow-convention.md](flow-convention.md) |
| Validator | `@Component` 선언. Port 주입 금지. 조회된 데이터를 파라미터로 받아 규칙만 검증 | [validator-convention.md](validator-convention.md) |
| Handler | `@Component` 선언. 여러 도메인 재사용 로직 또는 Flow → 타 도메인 Port의 ACL | [handler-convention.md](handler-convention.md) |
| Policy | 인터페이스 + 여러 구현체. `List<Policy>` 주입 + `supports(type)` 디스패치 | [policy-convention.md](policy-convention.md) |
| EventHandler | `@Component` 선언. `@TransactionalEventListener(phase = AFTER_COMMIT)`. 자기 도메인 Flow만 호출 | [event-handler-convention.md](event-handler-convention.md) |
| Mapper | `@Component` 선언. `mapper/{Entity}DtoMapper.kt`. UseCase 내 인라인 매핑 금지 | [mapper-convention.md](mapper-convention.md) |

---

## 책임 분배 순서

| 순서 | 책임 | 담당 객체 |
|------|------|-----------|
| 1 | 입력값 형식 검증 (필수값, 범위, 포맷) | Command의 `init` 블록 |
| 2 | 데이터 조회 | UseCase 또는 Flow에서 Port 조회 |
| 3 | 비즈니스 규칙 검증 | Validator (조회된 객체를 매개변수로 전달) |
| 4 | 도메인 행위 호출 (상태 변경) | Domain 객체 메서드 |
| 5 | 결과 변환 | Mapper 클래스 |

> 각 단계별 상세는 [use-case-convention.md](use-case-convention.md) "책임 분배 원칙" 섹션 참고

---

## Port Rules

- outbound port 인터페이스(Repository, 외부 서비스 추상)와 그 계약 데이터 클래스는 `port/` 서브디렉토리에 위치한다.
- `dto/`는 **controller ↔ UseCase 계약** (Command / Result). `port/`는 **application ↔ infrastructure 계약** (Repository, 외부 서비스 인터페이스, 그 인터페이스 전용 데이터 클래스).
- port 전용 데이터 클래스 판단 기준: "이 클래스가 port 인터페이스 시그니처에서만 의미를 가지는가?" → `port/`에 위치. 그렇지 않으면 `dto/` 또는 도메인 패키지에 위치.
- UseCase 클래스 자체가 inbound entry point 역할을 하므로 inbound port 인터페이스는 별도로 추출하지 않는다. (인터페이스 분리가 필요한 특수 케이스에만 예외 허용)

---

## Logging Rules

- 비즈니스 흐름 추적은 `common/log/LogExtension.kt`의 `logInfo` / `logWarn` / `logError` / `logDebug`를 사용한다.
- **상태 변경 성공 직후** UseCase 반환 직전에만 호출한다. 조회·검증 단계에서는 호출하지 않는다.
- 메시지 앞에 `[domain.action]` 태그를 붙여 액션을 식별한다 (예: `[skin.create]`).
- MDC에 이미 `tenantId` / `requestId`가 있으므로 메시지에 중복 포함하지 않는다.
- 개인정보·민감 데이터(비밀번호, 토큰 등)는 포함하지 않는다.

---

## Testing

- 신규 UseCase 또는 UseCase 구현(Service) 추가/변경 시 해당 변경셋에 단위 테스트를 반드시 함께 추가한다.
- UseCase 단위 테스트는 Kotest `BehaviorSpec` + Mockk 기반으로 작성하고, outbound Port/외부 연동은 mock으로 격리한다.
- 테스트 가독성을 위해 도메인별 `Fixture`(`object XxxFixture`)를 함께 작성해 요청/응답/도메인 생성을 재사용한다.
- 주요 플로우는 통합 테스트를 추가한다.

---

## Post-Work Verification

**가이드라인 문서를 참고했더라도 실제 생성된 코드에 반영되지 않은 부분이 있을 수 있다.** 구현 완료 후, 생성하거나 수정한 파일을 직접 읽어서 아래 각 하위 문서의 체크리스트를 하나씩 대조한다. 위반이 발견되면 즉시 수정하고 다시 검증한다.

각 문서 하단의 "체크리스트" 섹션을 참고한다.

- [use-case-convention.md](use-case-convention.md)
- [flow-convention.md](flow-convention.md)
- [validator-convention.md](validator-convention.md)
- [handler-convention.md](handler-convention.md)
- [policy-convention.md](policy-convention.md)
- [event-handler-convention.md](event-handler-convention.md)
- [mapper-convention.md](mapper-convention.md)
