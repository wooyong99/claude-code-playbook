# Layer Analysis Guide

각 레이어를 분석할 때 무엇을 읽고 어떤 패턴을 추출해야 하는지 정리한 가이드.

## 먼저: 로컬 레이어 맵을 만든다

이 가이드는 `domain`, `application`, `storage`, `external`, `app`를 **플레이북의 개념 레이어 이름**으로 사용한다.
하지만 실제 프로젝트는 전혀 다른 이름을 쓸 수 있다.

예시:

- `app` 대신 `api`, `presentation`, `bootstrap`
- `storage`와 `external` 대신 상위 `infra`, `infrastructure`
- `infra` 하위에 `storage`, `external`, `messaging`, `security`

따라서 분석 시작 전에 반드시 아래 두 단계를 먼저 수행한다.

1. 프로젝트의 **로컬 레이어 이름**을 식별한다.
2. 그 로컬 레이어를 플레이북의 **개념 레이어**에 매핑한다.

중요:

- 플레이북의 예시 레이어 이름과 **동일한 이름이 존재할 필요는 없다**.
- 이름이 달라도, 실제 코드에서 수행하는 **역할, 책임, 의존 방향, 포함된 클래스의 종류**를 기준으로 개념 레이어에 매핑한다.
- 즉, 매핑은 `이름 기반 분류`가 아니라 `역할 기반 분류`다.

예시 매핑 메모:

```text
로컬 레이어 맵
- bootstrap -> app
- core -> domain
- usecase -> application
- infrastructure/jpa -> storage
- infrastructure/client -> external
```

문서 생성 시에는 이 매핑을 숨기지 말고 README나 본문에 함께 드러낸다.

예를 들어:

- `bootstrap`, `api`, `presentation`은 표현 계층 역할이면 `app`으로 매핑할 수 있다.
- `usecase`, `service`, `application-service`는 응용 조합 책임이 강하면 `application`으로 매핑할 수 있다.
- `adapter`, `driven`, `infra`, `infrastructure`는 내부를 다시 쪼개 보고 `storage`, `external`, 기타 infrastructure 책임으로 분해할 수 있다.

## 구조 기준 선택

### 멀티 모듈 프로젝트

- `settings.gradle.kts` 기준으로 모듈 목록을 먼저 파악한다.
- 모듈명, 의존 관계, `src/main` 클래스 역할을 바탕으로 레이어별 **모듈**을 식별한다.
- 모듈 내부에 다시 하위 레이어가 섞여 있으면 패키지 기준으로 한 번 더 분해한다.
- 모듈 이름이 플레이북 예시와 달라도, 실제 책임이 같다면 같은 개념 레이어로 매핑한다.

### 단일 모듈 프로젝트

- `src/main/kotlin` 이하 디렉토리와 패키지 구조를 기준으로 레이어를 식별한다.
- 패키지명과 클래스 역할이 충돌하면 클래스 역할과 의존 방향을 더 우선한다.
- 디렉토리 이름이 낯설더라도, 실제 포함된 클래스 종류와 책임을 보고 레이어를 판정한다.

## 1. Domain

우선적으로 볼 대상:

- 로컬 레이어 맵에서 `domain`에 대응된 모듈 또는 디렉토리
- 순수 Kotlin 모델
- 도메인 예외
- 값 객체, 애그리거트, 팩토리, 상태 전이 메서드

확인할 패턴:

- 엔티티/값 객체를 어떻게 나누는가
- 불변식 검증이 어디에 있는가
- 도메인 예외 코드 구조가 있는가
- ID 참조 방식과 연관 규칙이 무엇인가

## 2. Application

우선적으로 볼 대상:

- 로컬 레이어 맵에서 `application`에 대응된 모듈 또는 디렉토리
- `UseCase`, `Service`, `Flow`, `Validator`, `Handler`, `Policy`, `Mapper`
- 트랜잭션 경계가 선언된 클래스
- Port 인터페이스

확인할 패턴:

- UseCase와 Flow를 분리하는가
- 입력 검증은 어디서 하는가
- 매핑 책임은 누가 가지는가
- 정책 객체나 전략 객체를 분리하는가

## 3. Storage

우선적으로 볼 대상:

- 로컬 레이어 맵에서 `storage`에 대응된 모듈 또는 디렉토리
- `Entity`, `Repository`, `RepositoryImpl`, `Adapter`
- JPA, Querydsl, jOOQ, MyBatis 관련 클래스
- persistence mapper 또는 entity converter

확인할 패턴:

- 저장소 adapter 구조가 있는가
- ORM 또는 쿼리 도구를 무엇으로 통일하는가
- Entity와 Domain 모델 변환 책임이 어디에 있는가
- 커스텀 조회 구현 패턴이 있는가

## 4. External

우선적으로 볼 대상:

- 로컬 레이어 맵에서 `external`에 대응된 모듈 또는 디렉토리
- 외부 API client
- adapter, dto, config, exception, error code
- HTTP client 설정

확인할 패턴:

- 외부 연동을 adapter로 감싸는가
- DTO 네이밍과 계층 분리가 어떻게 되는가
- 예외 매핑과 로깅 규칙이 있는가
- 사용하는 HTTP client가 무엇인가

## 5. App

우선적으로 볼 대상:

- 로컬 레이어 맵에서 `app`에 대응된 모듈 또는 디렉토리
- controller
- request/response DTO
- exception handler
- API path와 버전 규칙

확인할 패턴:

- controller가 command/query 변환만 담당하는가
- request DTO에 validation annotation이 어떻게 붙는가
- 공통 응답 포맷이 있는가
- 예외 처리를 글로벌하게 통합하는가

## 6. Infrastructure 또는 기타 상위 레이어

어떤 프로젝트는 `storage`, `external`이 독립 레이어가 아니라 `infra` 또는 `infrastructure` 하위에 함께 존재한다.
이 경우 상위 레이어를 한 번에 문서화하지 말고, 하위 책임 단위로 다시 나눈다.

우선적으로 볼 대상:

- `infra`, `infrastructure`, `adapter`, `driven`
- 하위의 `storage`, `external`, `client`, `persistence`, `messaging`, `security`

확인할 패턴:

- 상위 infrastructure 레이어가 어떤 하위 책임으로 쪼개지는가
- persistence와 external integration이 분리되는가
- 공통 설정과 구현체가 어떻게 배치되는가
- 하위 전략 문서를 플레이북의 어느 레이어에 대응시킬지

문서화 규칙:

- `infra`가 상위 묶음이면 README에서 먼저 구조를 설명한다.
- 그 다음 실제 내용은 `storage`, `external` 또는 다른 하위 전략 단위로 나눠 설명한다.
- 플레이북 표준 구조와 1:1로 맞지 않을 경우, "이 프로젝트의 infrastructure 레이어는 플레이북의 storage/external을 함께 포함한다" 같은 매핑 설명을 적는다.

## 분석 메모 형식

레이어별 분석 결과는 최소 아래 형태로 메모한다.

```text
[LAYER] 분석 결과
전략: {핵심 전략명}
로컬 레이어명: {프로젝트에서 실제로 쓰는 이름}
개념 레이어 매핑: {domain/application/storage/external/app/infrastructure 중 대응}
근거:
- {관찰 1}
- {관찰 2}
컴포넌트:
- {역할}: {예시 클래스}
- {역할}: {예시 클래스}
불확실한 부분:
- {있다면 기록, 없으면 없음}
```
