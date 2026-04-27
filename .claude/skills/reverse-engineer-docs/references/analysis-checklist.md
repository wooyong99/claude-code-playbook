# 계층별 분석 체크리스트

코드베이스 분석 시 각 계층에서 수집해야 할 정보 목록.
Explore 에이전트에게 위임할 때 이 체크리스트를 기준으로 탐색 범위를 지정한다.

---

## 공통: 전체 프로젝트 스캔

### 모듈 구조 파악
```bash
# 멀티모듈 구조 확인
cat settings.gradle.kts  # or settings.gradle
# 각 모듈의 의존성 확인
find . -name "build.gradle.kts" -not -path "*/build/*" | xargs grep -l "implementation"
```

### 기술 스택 식별
- [ ] 언어 및 버전 (`kotlin`, `java`, `typescript` 등)
- [ ] 프레임워크 (`spring-boot`, `ktor`, `nestjs` 등)
- [ ] ORM/쿼리 (`spring-data-jpa`, `querydsl`, `exposed`, `prisma` 등)
- [ ] 테스트 프레임워크 (`kotest`, `junit5`, `jest` 등)
- [ ] 인증 방식 (`spring-security`, `jwt`, `oauth2` 등)

---

## API 계층 (Controller / Router)

### 탐색 경로
```bash
find . -type f -name "*Controller*" -o -name "*Router*" -o -name "*Handler*" | grep -v test | grep -v build
```

### 수집 항목
- [ ] 클래스명 패턴 (suffix: `Controller`, `Router`, `Handler` 등)
- [ ] 어노테이션 패턴 (`@RestController`, `@RequestMapping`, `@PostMapping` 등)
- [ ] Request/Response DTO 위치 및 명명 패턴
- [ ] 예외 처리 방식 (`@ControllerAdvice`, `@ExceptionHandler` 위치)
- [ ] API 경로 패턴 (`/api/v1/...`, `/v1/...` 등)
- [ ] 인증 필터/미들웨어 위치

### 확인 질문
- 요청 검증은 어디서? (Controller 파라미터 vs Validator 클래스)
- 응답 포맷은 공통화되어 있는가? (공통 `ApiResponse<T>` wrapper 등)
- 예외 응답 포맷은?

---

## 애플리케이션 계층 (Service / UseCase)

### 탐색 경로
```bash
find . -type f -name "*UseCase*" -o -name "*Service*" -o -name "*Command*" | grep -v test | grep -v build | grep -v "node_modules"
```

### 수집 항목
- [ ] 클래스명 패턴 (`UseCase`, `Service`, `ApplicationService` 등)
- [ ] Command/Query 객체 위치 및 패턴
- [ ] 트랜잭션 경계: `@Transactional`이 어느 계층에 있는가
- [ ] 비즈니스 로직 분해 방식 (Flow, Handler, Policy 등 서브 컴포넌트 여부)
- [ ] 도메인 Port 인터페이스 위치
- [ ] 이벤트 발행 패턴 (ApplicationEventPublisher 등)

### 확인 질문
- UseCase와 Service를 모두 쓰는가, 아니면 둘 중 하나만 쓰는가?
- CQRS 패턴(Command/Query 분리)이 적용되어 있는가?
- Flow/Handler 같은 중간 계층 컴포넌트가 있는가?

---

## 도메인 계층 (Entity / Aggregate / ValueObject)

### 탐색 경로
```bash
find . -path "*/domain/*" -type f -name "*.kt" | grep -v test | grep -v build
find . -type f -name "*Entity*" -o -name "*Aggregate*" -o -name "*ValueObject*" | grep -v test | grep -v build
```

### 수집 항목
- [ ] 엔티티 클래스 구조 (JPA Entity vs 순수 도메인 객체 분리 여부)
- [ ] 도메인 이벤트 패턴 (내부 이벤트, 외부 이벤트)
- [ ] 집계(Aggregate) 경계 — 어떤 클래스 그룹이 함께 변경되는가
- [ ] 값 객체(ValueObject) 패턴
- [ ] 도메인 예외 클래스 계층 구조
- [ ] Port 인터페이스 정의 위치

### 확인 질문
- JPA Entity와 도메인 모델이 분리되어 있는가?
- 도메인 객체가 비즈니스 메서드를 포함하는가, 아니면 빈약한 모델(anemic)인가?
- 팩토리 메서드 패턴을 사용하는가?

---

## 영속성 계층 (Repository / Storage)

### 탐색 경로
```bash
find . -type f -name "*Repository*" -o -name "*Dao*" -o -name "*Adapter*" | grep -v test | grep -v build
find . -path "*/storage/*" -type f | grep -v test | grep -v build
```

### 수집 항목
- [ ] Repository 인터페이스 vs 구현체 분리 방식
- [ ] JPA 사용 시: `JpaRepository` 상속 여부, `@Query` vs QueryDSL 비중
- [ ] QueryDSL 사용 시: Q클래스 위치, Custom Repository 구조
- [ ] 페이징 처리 방식 (`Pageable` vs 커서 기반)
- [ ] 영속성 Entity와 도메인 Entity 매핑 방식 (Storage Adapter 패턴 여부)

### 확인 질문
- 도메인 Port(인터페이스)의 구현체가 이 계층에 있는가?
- 단순 Spring Data JPA Repository만 사용하는가, 아니면 Custom 구현이 많은가?
- DDL 관리 방식은? (Flyway, Liquibase, JPA DDL-auto, 수동)

---

## 외부 연동 계층 (Client / Adapter)

### 탐색 경로
```bash
find . -type f -name "*Client*" -o -name "*Feign*" -o -name "*Adapter*" | grep -v test | grep -v build
find . -path "*/external/*" -type f | grep -v test | grep -v build
find . -path "*/client/*" -type f | grep -v test | grep -v build
```

### 수집 항목
- [ ] HTTP 클라이언트 방식 (`FeignClient`, `RestTemplate`, `WebClient`, `RestClient`)
- [ ] 외부 API 요청/응답 DTO 위치 및 패턴
- [ ] 재시도/서킷브레이커 패턴 (Resilience4j, Spring Retry)
- [ ] 외부 API 예외 처리 방식 (커스텀 예외 변환 여부)
- [ ] Mock/Stub 전략 (테스트용 Mock Adapter 여부)
- [ ] API 클라이언트 설정 방식 (timeout, retry 설정 위치)

---

## 크로스커팅: 보안

```bash
find . -name "*Security*" -o -name "*Auth*" -o -name "*Jwt*" | grep -v test | grep -v build
find . -name "SecurityConfig*" | grep -v test | grep -v build
```

### 수집 항목
- [ ] `SecurityFilterChain` 또는 보안 설정 클래스 위치
- [ ] JWT 파싱/검증 로직 위치
- [ ] 인증 컨텍스트 접근 방식 (`SecurityContextHolder`, 커스텀 어노테이션 등)
- [ ] 역할(Role)/권한(Permission) 구조
- [ ] CORS 설정

---

## 크로스커팅: 로깅

```bash
find . -name "logback*.xml" -o -name "log4j*.xml" -o -name "logback*.yml"
grep -r "MDC\." --include="*.kt" --include="*.java" -l | head -10
grep -r "LoggerFactory\|@Slf4j\|private val log\|private val logger" --include="*.kt" --include="*.java" -l | head -10
```

### 수집 항목
- [ ] 로그 설정 파일 위치 및 형식 (JSON vs 텍스트)
- [ ] MDC 키 목록 및 설정 위치
- [ ] 공통 로깅 필터/인터셉터 위치
- [ ] 로그 레벨별 사용 패턴

---

## 크로스커팅: 트랜잭션

```bash
grep -r "@Transactional" --include="*.kt" --include="*.java" -l | grep -v test | grep -v build
grep -r "@Transactional" --include="*.kt" --include="*.java" | grep -v test | grep -v build | head -30
```

### 수집 항목
- [ ] `@Transactional` 선언 위치 분포 (계층별)
- [ ] 읽기 전용 트랜잭션 패턴 (`readOnly = true`)
- [ ] 트랜잭션 전파 레벨 커스터마이즈 사례
- [ ] 도메인 이벤트와 트랜잭션 관계 (`@TransactionalEventListener`)

---

## 크로스커팅: 동시성/성능

```bash
grep -r "@Lock\|@Version\|@Cacheable\|@Async" --include="*.kt" --include="*.java" -l | grep -v test | grep -v build
grep -r "EntityGraph\|fetchType\|BatchSize\|fetch join" --include="*.kt" --include="*.java" | grep -v test | head -20
```

### 수집 항목
- [ ] 낙관적 잠금 (`@Version`) 사용 엔티티
- [ ] 비관적 잠금 (`@Lock`) 사용 Repository 메서드
- [ ] 캐시 어노테이션 사용 위치 (`@Cacheable`, `@CacheEvict`)
- [ ] N+1 방지 패턴 (`@EntityGraph`, `fetch join`, `batch_fetch_size`)
- [ ] 비동기 처리 (`@Async`, Coroutine, `CompletableFuture`) 사용 위치

---

## 분석 완료 기준

각 계층에서 다음을 확보하면 해당 계층의 문서를 작성할 준비가 된 것으로 본다:

- [ ] 실제 클래스명 예시 3개 이상
- [ ] 핵심 어노테이션/패턴 파악
- [ ] 의존 관계 방향 파악
- [ ] 대표 코드 스니펫 1~2개 (파일 경로 포함)

파악하지 못한 항목은 `[분석 필요: {이유}]`로 표시하고 문서 작성을 진행한다.
