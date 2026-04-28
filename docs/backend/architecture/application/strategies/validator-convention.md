# Validator 컨벤션

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)에서 **Rule Checker** 역할을 담당하는 `Validator` 컴포넌트의 컨벤션이다.
> 다른 프로젝트에서는 동일한 역할을 `Guard`, `Specification`, `DomainRule` 등으로 구현할 수 있다.

## 보편 개념

**Rule Checker / Specification**은 조회된 데이터를 받아 비즈니스 규칙을 검증한다. 인프라 의존성이 없어야 단독으로 테스트 가능하며, 조회 책임은 호출 측(진입점 또는 흐름 단위)에 있다. 같은 입력으로 판단 가능한 규칙을 하나의 메서드로 묶고, 중간 조회가 필요한 규칙은 별도 메서드로 분리한다.

---

## 원칙

- Validator는 순수하다. Port 의존성이 없어야 단독으로 테스트 가능하고, 숨겨진 조회 로직이 생기지 않는다.
- 조회는 호출 측 책임이다. Validator가 데이터를 스스로 조회하면 조회 비용과 실패 지점이 가려진다.
- 검증 규칙의 경계는 "같은 입력으로 판단 가능한가"로 정한다. 중간에 외부 조회가 필요하면 메서드를 나눠 흐름을 명시적으로 드러낸다.
- Validator를 묶는 Validator는 만들지 않는다. UseCase 전용 Validator는 조회 은닉 또는 응집도 없는 묶음 중 하나로 귀결된다.

---

## 핵심 규칙

**Validator는 Port를 주입받지 않는다. 조회된 객체를 매개변수로 전달받아 규칙만 검증한다.**

조회 책임이 Validator 안에 숨지 않고 호출 측(Flow/UseCase) 코드에 명시적으로 드러나도록 한다.

---

## 예시

```kotlin
// ❌ Validator가 Port를 주입받아 직접 조회
@Component
class FileValidator(private val tenantRepository: TenantRepository) {
    fun validateUploadPermission(tenantId: String, fileName: String) {
        val tenant = tenantRepository.findById(tenantId)  // 조회 책임 혼입
            ?: throw CoreException(TenantErrorCode.TENANT_NOT_FOUND)
    }
}

// ✅ Port 의존성 없음 — 조회된 객체를 받아 규칙만 검증
@Component
class FileValidator {
    fun validateUploadPermission(tenant: Tenant, fileName: String) {
        require(tenant.isFileUploadEnabled) { "파일 업로드가 허용되지 않은 테넌트입니다." }
        val extension = fileName.substringAfterLast(".", "").lowercase()
        if (extension !in ALLOWED_EXTENSIONS) {
            throw CoreException(FsNodeErrorCode.INVALID_REQUEST, "허용되지 않는 파일 형식입니다.")
        }
    }

    fun validateNoDuplicate(existing: FsNode?, fileName: String) {
        if (existing != null) {
            throw CoreException(FsNodeErrorCode.INVALID_REQUEST, "같은 경로에 동일한 파일명이 존재합니다.")
        }
    }

    companion object {
        private val ALLOWED_EXTENSIONS = setOf("jpg", "jpeg", "png", "gif", "svg", "webp", "pdf")
    }
}
```

---

## 추출 판단 기준

- 비즈니스 정책 검증 (허용 확장자, 권한 등) → Validator
- 여러 Flow/UseCase에서 동일한 검증을 공유 → Validator
- 검증에 필요한 데이터가 있는 경우 → Flow/UseCase에서 Port 조회 후 Validator에 매개변수로 전달
- 단순 존재 여부 확인 → Flow/UseCase에서 직접 Port 조회 (Validator 불필요)

---

## UseCase 단위 Validator 안티패턴

**규칙: 여러 Validator를 묶는 "UseCase 단위 Validator"(예: `RegisterTenantValidator`)는 만들지 않는다.**

UseCase에 여러 Validator 호출이 모여 "비즈니스 흐름이 검증 코드에 가려진다"고 느낄 때, 그것들을 묶어 UseCase 전용 Validator로 추출하고 싶은 유혹이 생긴다. 이는 `*ValidateFlow` 안티패턴(`flow-convention.md` 참고)과 본질적으로 같은 구조적 문제를 만든다.

### 왜 만들지 않는가

UseCase Validator는 결국 아래 세 형태 중 하나로 수렴하는데, 모두 기존 원칙과 충돌한다.

- 내부에서 Port를 주입받아 조회 수행 → 조회 책임 은폐 — Validator 핵심 규칙 위반 (ValidateFlow 안티패턴 재현)
- Port 없이 모든 조회 결과를 파라미터로 수신 → 시그니처 비대화, 응집도 없는 "묶음"만 남음
- 다른 Validator들을 주입받아 호출 순서만 지휘 → 실질 규칙 없는 단순 위임 — 간접 레벨만 +1

### "흐름이 안 보인다"의 올바른 해법

UseCase에 검증 코드가 많아 흐름이 가려지는 것은 **검증이 Flow로 내려가야 한다는 신호**인 경우가 많다. `flow-convention.md`의 "전제 조건 vs Flow 내부 조건" 분류를 따른다.

- **전제 조건**(UseCase 전체 실행의 선결 조건) → UseCase에서 조회·검증
- **Flow 내부 조건**(특정 업무 흐름 내부에서만 의미 있는 조건) → Flow에서 조회·검증

예를 들어 테넌트 등록 UseCase의 "슬러그 형식/예약/중복", "이메일 중복", "비밀번호 정책", "사업자번호" 검증은 모두 **Tenant 생성 Flow 내부 조건**이다. 해당 Flow로 이관하면 UseCase에는 선행 조건인 "약관 동의 검증"만 남아 흐름이 드러난다.

```kotlin
// ✅ 검증을 Flow로 이관한 후 UseCase — 조합만 남음
@Service
class RegisterTenantUseCase(
    private val termsRepository: TermsRepository,
    private val termsRequiredValidator: TermsRequiredValidator,
    private val tenantOnboardingFlow: TenantOnboardingFlow,
    private val tenantDtoMapper: TenantDtoMapper,
) {
    fun register(command: RegisterTenant.Command): RegisterTenant.Result {
        // 1. UseCase 전제 조건 — 약관 동의 (Tenant 생성과 독립된 선행 조건)
        val requiredVersionIds = termsRepository.findActiveRequiredVersions()
            .map { it.id }.toSet()
        val activeVersions = termsRepository.findActiveVersionsByIds(
            command.agreedTermsVersionIds.toSet()
        )
        termsRequiredValidator.validate(
            requiredVersionIds = requiredVersionIds,
            agreedVersionIds = command.agreedTermsVersionIds.toSet(),
            activeVersions = activeVersions,
        )

        // 2. Tenant 생성 Flow에 위임 — slug/email/password/businessRegNo 검증은 Flow 내부 책임
        val result = tenantOnboardingFlow.execute(command)
        return tenantDtoMapper.toResult(result.tenant, result.shop, result.account)
    }
}
```

---

## 퍼블릭 메서드 경계 기준

**규칙: Validator 퍼블릭 메서드는 "같은 입력·같은 컨텍스트에서 판단 가능한가"로 경계를 정한다. "함께 호출되는가"는 기준이 아니다.**

세부 규칙을 단순히 묶어 하나의 `validate()`로 만드는 것도, 모든 규칙을 별도 퍼블릭 메서드로 쪼개는 것도 기준 없이 하면 임의적이다. 아래 기준으로 판단한다.

- 동일 입력만으로 판단되는 여러 규칙 → **하나의 메서드로 통합**, 세부 규칙은 내부 `private` 메서드로 분리 (예: `TermsRequiredValidator.validate` — 3개 Set 입력으로 "활성 버전 여부 + 필수 동의 여부"를 한 번에 검증)
- 규칙 사이에 Port / Policy 조회가 끼어드는 경우 → **메서드 분리** — 호출 순서와 중간 조회가 UseCase/Flow 흐름에 드러나야 함 (예: `SlugValidator.validateFormat` → `slugReservedPolicy.isReserved()` 조회 → `SlugValidator.validateNotReserved`)
- 조건에 따라 일부 규칙만 적용되는 경우 → **메서드 분리** — 생성·수정 등 컨텍스트별 선택 가능 (예: 생성 시 모든 규칙 vs 수정 시 일부 규칙)

### 판단 예시

```kotlin
// ✅ 통합 — 조회 없이 입력만으로 판단 가능한 복수 규칙
@Component
class TermsRequiredValidator {
    fun validate(
        requiredVersionIds: Set<Long>,
        agreedVersionIds: Set<Long>,
        activeVersions: List<TermsVersion>,
    ) {
        validateAllAgreedAreActive(agreedVersionIds, activeVersions)
        validateAllRequiredAreAgreed(requiredVersionIds, agreedVersionIds)
    }

    private fun validateAllAgreedAreActive(...) { ... }
    private fun validateAllRequiredAreAgreed(...) { ... }
}

// ✅ 분리 — 규칙 사이에 Policy 조회가 끼어듦
@Component
class SlugValidator {
    fun validateFormat(slug: String) { ... }
    fun validateNotReserved(slug: String, reserved: Boolean) { ... }
}

// 호출 측에서 중간 조회가 흐름에 드러남
slugValidator.validateFormat(command.slug)
val reserved = slugReservedPolicy.isReserved(command.slug)
slugValidator.validateNotReserved(command.slug, reserved)
```

### 피해야 할 패턴

```kotlin
// ❌ 외부 조회가 필요한 규칙을 한 시그니처 뒤로 숨김
//    - 호출 측에서 Policy 조회가 보이지 않아 흐름이 감춰짐
//    - 형식 검증 실패 여부와 무관하게 Policy 조회가 선행 실행됨 (인자 평가)
slugValidator.validate(
    slug = command.slug,
    reserved = slugReservedPolicy.isReserved(command.slug),
)
```

---

## Validator 호출 위치

Validator는 Flow 또는 UseCase 어디서든 호출할 수 있다. 검증의 범위로 위치를 결정한다.

- 전체 UseCase 실행의 전제 조건 → UseCase에서 직접 호출
- 특정 Flow 단계의 내부 조건 → 해당 Flow 내부에서 호출

**Port 조회가 필요한 경우**: 호출 측(UseCase 또는 Flow)에서 먼저 Port를 통해 데이터를 조회한 뒤 Validator에 파라미터로 전달한다. Validator 자체에 Port를 주입해 "ValidateFlow"처럼 사용하는 것은 안티패턴이다 (`flow-convention.md` 참고).

```kotlin
// UseCase가 선행 조건 검증을 담당하는 예
@Service
class RegisterTenantUseCase(
    private val termsRepository: TermsRepository,        // UseCase에서 직접 조회
    private val termsRequiredValidator: TermsRequiredValidator,
    private val tenantCreateFlow: TenantCreateFlow,
    // ...
) {
    @Transactional
    fun register(command: RegisterTenant.Command) {
        // 1. UseCase에서 검증에 필요한 데이터 조회
        val requiredVersionIds = termsRepository.findActiveRequiredVersions()
            .map { it.version.id }.toSet()
        val activeVersions = termsRepository.findActiveVersionsByIds(
            command.agreedTermsVersionIds.toSet()
        )

        // 2. Validator에 조회된 데이터 전달 — 규칙 검증만 담당
        termsRequiredValidator.validate(
            requiredVersionIds = requiredVersionIds,
            agreedVersionIds = command.agreedTermsVersionIds.toSet(),
            activeVersions = activeVersions,
        )

        // 3. 검증 통과 후 Flow 조합
        tenantCreateFlow.execute(...)
        // ...
    }
}
```

---

## 체크리스트

- [ ] `@Component`로 선언했는가?
- [ ] Port를 주입받지 않는가?
- [ ] 검증에 필요한 데이터를 호출 측(Flow/UseCase)에서 조회 후 매개변수로 받는가?
- [ ] 비즈니스 정책 상수를 `companion object`에 선언했는가?
- [ ] 여러 Validator를 묶는 UseCase 단위 Validator(`{UseCase}Validator`)를 만들지 않았는가?
- [ ] 퍼블릭 메서드가 "동일 입력으로 판단 가능한 규칙은 통합, 중간 Port/Policy 조회가 필요한 규칙은 분리" 기준을 따르는가?
