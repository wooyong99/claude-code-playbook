# API 컨벤션 (App Layer)

---

## 핵심 규칙

**Controller는 Request DTO를 받아 Extension 함수로 Command 변환 → UseCase 호출 → `BaseResponse<Result>` 응답만 수행한다. application 계층의 Command를 `@RequestBody`로 직접 받지 않는다.**

Request/Response DTO는 Spring 검증 어노테이션만 포함하고 변환 로직이나 비즈니스 규칙을 두지 않는다. 변환 로직은 `{Domain}DtoExtension.kt`의 Extension 함수가 담당하며, `dto/` 패키지가 아닌 Controller와 같은 도메인 패키지 레벨에 위치한다.

> 예외 응답 처리는 [exception-handling-convention.md](exception-handling-convention.md) 참고

---

## 네이밍 규칙

- **Controller 클래스**: `{Domain}Controller` (admin 모듈: `Admin{Domain}Controller`) — `OrderController`, `AdminUserController`
- **Request 파일**: `{Domain}Requests.kt` (복수). 1개면 단수형 — `OrderRequests.kt`, `OrderRequest.kt`
- **Response 파일**: `{Domain}Responses.kt` (복수). 1개면 단수형 — `OrderResponses.kt`
- **Extension 파일**: `{Domain}DtoExtension.kt` — `OrderDtoExtension.kt`
- **Request 클래스**: `{Action}{Entity}Request` — `CreateOrderRequest`, `UploadFileRequest`
- **Response 클래스**: `{Action}{Entity}Response` — `GetOrderDetailResponse`
- **Request → Command 변환**: `request.toCommand()` — `createOrderRequest.toCommand()`
- **Result → Response 변환**: `result.toResponse()`

---

## 경로 네이밍 (요약)

Controller의 `@RequestMapping` 경로는 **복수형 명사 + kebab-case + `/api/v{N}/` prefix** 를 따른다.

> URL 네이밍 · Plural 규칙 · ID 위치(Path vs Query) · 페이지네이션 표준 등 REST API 설계 상세는 [rest-design-convention.md](rest-design-convention.md) 참고

---

## Controller 구성

**규칙: Controller는 비즈니스 로직 없이 Request → Command 변환 → UseCase 호출 → 응답 반환만 수행한다.**

```kotlin
@RestController
@RequestMapping("/api/v1/{도메인}s")
class {Domain}Controller(
    private val useCase: {Action}{Entity}UseCase,
) {
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long): ResponseEntity<BaseResponse<Get{Entity}.Result>> {
        val result = useCase.getDetail(id)
        return ResponseEntity.ok(BaseResponse.success(result))
    }

    @PostMapping
    fun create(
        @Valid @RequestBody request: Create{Entity}Request,
    ): ResponseEntity<BaseResponse<Create{Entity}.Result>> {
        val result = useCase.create(request.toCommand())
        return ResponseEntity.status(HttpStatus.CREATED).body(BaseResponse.success(result))
    }
}
```

- `@RestController` 사용
- 공통 경로는 클래스 레벨 `@RequestMapping`으로 선언
- `@Valid`를 Request DTO에 적용한다
- 화면 렌더링(asset 서빙, SSR 등)은 별도 Controller로 분리
- `@CurrentUser` 등 인프라 관심사는 Controller에서 주입받아 Extension에 전달

---

## 응답 형식

**규칙: 모든 REST API 응답은 `BaseResponse<T>`로 래핑한다.**

```
{ "data": { ... } }                                    // 성공 (data, error 상호 배타)
{ "error": { "code": "...", "message": "..." } }       // 실패
```

- `null` 필드는 JSON에 미포함 (`@JsonInclude(NON_NULL)`)
- 성공/실패 판단은 HTTP 상태 코드로 — `success` 필드 없음

> 예외 응답 포맷과 처리 흐름은 [exception-handling-convention.md](exception-handling-convention.md) 참고

---

## Request DTO

**규칙: Request DTO는 Spring 검증 어노테이션만 포함한다. 변환 로직이나 비즈니스 규칙을 포함하지 않는다.**

- Controller는 Command를 직접 받지 않음 — Request DTO → Command 변환은 Extension 함수
- Spring 검증 어노테이션 포함 — `@NotBlank`, `@Size`, `@Email`, `@Min`, `@Max`
- 변환 로직(`toCommand()`) 금지 — Extension 함수로 분리
- 같은 도메인은 한 파일로 — `{Domain}Requests.kt`

```kotlin
// {Domain}Requests.kt
data class Create{Entity}Request(
    @field:NotBlank(message = "이름은 비어있을 수 없습니다.")
    val name: String,

    @field:Size(max = 1000)
    val description: String?,

    val parentId: Long?,
)
```

---

## Response DTO

**규칙: 기본은 application 계층의 `Result`를 `BaseResponse<Result>`로 직접 반환한다. API 전용 포맷이 필요한 경우에만 Response DTO를 생성한다.**

- Result를 그대로 반환 가능 → Response DTO **불필요**
- API 전용 날짜/숫자 포맷 (`@JsonFormat`) → Response DTO 생성
- Result의 일부 필드만 노출 → Response DTO 생성
- 여러 Result 조합 응답 → Response DTO 생성
- API 버전 간 응답 차이 → Response DTO 생성

생성 시 `{Domain}Responses.kt` 한 파일에 모아서 정의한다.

---

## 매핑 (Extension 함수)

**규칙: Request → Command, Result → Response 변환은 `{Domain}DtoExtension.kt`의 Extension 함수로 작성한다. 파일 위치는 `dto/` 밖 도메인 패키지 레벨이다.**

변환은 Controller 경계의 관심사이지 DTO 자신의 책임이 아니다. DTO 정의(`dto/`)와 변환 로직을 분리하여 Controller와 같은 레벨에 둔다.

```kotlin
// {Domain}DtoExtension.kt  (controller/{domain}/ 바로 아래)
fun Create{Entity}Request.toCommand() = Create{Entity}.Command(
    name = name,
    description = description,
    parentId = parentId,
)

fun Get{Entity}.Result.toResponse() = Get{Entity}Response(
    id = id,
    name = name,
    formattedAt = createdAt.format(DateTimeFormatter.ofPattern("yyyy-MM-dd")),
)
```

> `infra:storage`의 `{Entity}Extension.kt` 패턴과 동일한 원칙 (변환은 소유자가 아닌 경계 계층의 책임).


---

## 금지 사항

- Controller가 application 계층의 Command를 `@RequestBody`로 직접 받지 않는다.
- Request DTO에 `toCommand()` 등의 변환 로직을 포함하지 않는다.
- Request DTO에 비즈니스 규칙 검증(DB 조회 등)을 포함하지 않는다 — Spring 어노테이션만.
- Controller에 비즈니스 로직을 두지 않는다 — UseCase 위임만.
- `{Domain}DtoExtension.kt`를 `dto/` 패키지 안에 두지 않는다.
- Response DTO를 불필요하게 생성하지 않는다 — 기본은 `Result` 직접 반환.
- Controller에서 개별 `try-catch`로 예외를 처리하지 않는다 ([exception-handling-convention.md](exception-handling-convention.md) 참고).

---

## 체크리스트

### Controller
- [ ] `@RestController` + 클래스 레벨 `@RequestMapping` 구조인가?
- [ ] 클래스명이 `{Domain}Controller` (admin 모듈은 `Admin{Domain}Controller`) 형식인가?
- [ ] Command를 직접 `@RequestBody`로 받지 않는가?
- [ ] `@Valid`가 Request DTO에 적용됐는가?
- [ ] 비즈니스 로직 없이 `Request → Command → UseCase → 응답` 흐름만 있는가?
- [ ] 모든 응답이 `BaseResponse<T>`로 래핑됐는가?

### Request
- [ ] Spring 검증 어노테이션만 포함하고 변환 로직(`toCommand()`)이 없는가?
- [ ] 같은 도메인의 Request DTO가 `{Domain}Requests.kt` 한 파일에 모여 있는가?

### Response
- [ ] `Result`를 `BaseResponse<Result>`로 직접 반환하는가?
- [ ] Response DTO를 생성했다면 API 전용 포맷이 필요한 경우인가?
- [ ] Response DTO가 `{Domain}Responses.kt` 한 파일에 모여 있는가?

### 매핑
- [ ] `{Domain}DtoExtension.kt`가 `dto/` 밖 도메인 패키지 레벨에 있는가?
- [ ] Request → Command, Result → Response 변환이 모두 Extension 함수로 작성됐는가?

### URL
- [ ] URL 설계가 [rest-design-convention.md](rest-design-convention.md)의 체크리스트를 만족하는가? (복수형·kebab-case·ID 위치·페이지네이션 포함)
