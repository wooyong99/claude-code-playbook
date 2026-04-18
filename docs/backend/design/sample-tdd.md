# 테넌트 온보딩 기술설계문서 (TDD)

> 작성일: 2026-04-12
> 상태: Draft
> 대상 모듈: `:app:backoffice`, `:core:application`, `:core:domain`, `:infra:storage`

---

## 1. 설계 배경 및 목적

### 1.1 배경

본 플랫폼은 여러 독립 쇼핑몰(테넌트)이 하나의 시스템에서 운영되는 SaaS 구조다.
서비스 진입을 위해서는 **테넌트 가입(온보딩)** 이 선행되어야 하며, 이 시점에 다음이 동시에 성립해야 한다.

- 플랫폼에 신규 테넌트 공간이 생성되고,
- 해당 테넌트의 **기본 샵**이 준비되고,
- 최초 관리자(오너)의 **계정·멤버십**이 생성되고,
- 온보딩 시점에 활성 중인 **필수 약관에 대한 동의 기록**이 남아야 한다.

이 네 가지는 하나라도 누락되면 테넌트가 "반쯤 생성된" 비정상 상태로 남으므로, 단일 원자 단위로 처리되어야 한다.

### 1.2 설계 목표

1. **원자적 온보딩**: Tenant/Shop/Account/Membership/TermsAgreement 생성이 **단일 트랜잭션**으로 All-or-Nothing.
2. **비인증 공개 엔드포인트**: 테넌트-리스 환경에서 호출 가능해야 하며, `X-Tenant-ID` 헤더 없이도 동작.
3. **법적 증빙 보장**: 약관 동의 기록은 **동의 시점에 활성이던 버전(스냅샷)** 을 참조하고, 이후 버전 변경에 영향 받지 않는다.
4. **도메인 순수성 유지**: 온보딩 과정의 오케스트레이션은 Application 계층이 담당하고, 도메인 계층에는 "Registration" 같은 통합 애그리거트를 두지 않는다.
5. **멀티테넌트 규칙 준수**: CLAUDE.md의 데이터 분류(테넌트/플랫폼/동의 스냅샷) 및 예약 slug 정책을 위반하지 않는다.

### 1.3 설계 비목표

- **이메일/휴대폰 본인 인증**: 본 설계는 가입 즉시 `ACTIVE`로 전환. 인증 절차는 추후 과제.
- **관리자(오너) 권한 이관 / 다중 오너**: 테넌트당 오너 1명 고정. 이관 API 미제공.
- **CAPTCHA·봇 차단**: 공개 엔드포인트의 남용 방지는 본 TDD 범위 밖.
- **결제·플랜 선택**: 유료 플랜은 온보딩 이후 별도 업그레이드로 처리.
- **테넌트 탈퇴(WITHDRAWN) 후 데이터 파기**: 상태 전이만 다루며 물리 삭제/익명화 정책은 별도.
- **예약 발효 약관(scheduled effectiveFrom)**: 발행 시점 = 발효 시점만 지원.

### 1.4 기술적 제약사항

- **아키텍처**: Clean Architecture + DDD, 단방향 의존(`app → application → domain ← infra`). 도메인 계층에 Spring/JPA 누출 금지.
- **멀티테넌트**: 모든 영속 도메인은 `tenantId` 필드 보유. 플랫폼 데이터는 `tenantId=PLATFORM` 예약 상수 사용.
- **테넌트-리스 엔드포인트**: `TenantInterceptor` 화이트리스트로 관리. 온보딩 경로는 `X-Tenant-ID` 면제.
- **API 응답**: `BaseResponse<T>`로 래핑, `GlobalExceptionHandler`로 통합 예외 처리.
- **DB**: MySQL 8.0 (운영), H2 (테스트). 단일 DB + Row-level tenancy.
- **비기능**: 가입 요청은 실시간 응답이며 외부 연동 없음 → 평균 응답 1초 이내 목표.

---

## 2. 현행 시스템 분석

### 2.1 관련 도메인 구조 (신규 생성)

현 프로젝트에는 테넌트/샵/계정/약관 도메인이 존재하지 않는다. 본 TDD에서 **최초 도입**된다.

```
Tenant (1) ──→ (N) Shop               [tenantId로 참조]
Tenant (1) ──→ (N) Account            [tenantId로 참조]
Account (1) ──→ (N) Membership        [accountId로 참조]
Membership ──→ Tenant                 [tenantId]
Membership ──→ Shop (nullable)        [targetId, targetType=SHOP일 때]
Terms (1) ──→ (N) TermsVersion        [termsId]
TermsAgreement ──→ Account, TermsVersion  [accountId, termsVersionId]

(Terms, TermsVersion = tenantId=PLATFORM 플랫폼 데이터)
(TermsAgreement = tenantId=<tenant slug> 테넌트 데이터, 플랫폼 데이터 참조 후 불변)
```

### 2.2 현재 처리 흐름 (신규)

```
[Visitor] POST /api/v1/tenants
   ↓
:app:backoffice
   TenantOnboardingController
   ├─ @Valid RegisterTenantRequest
   └─ toCommand() → RegisterTenantCommand
   ↓
:core:application
   TenantOnboardingUseCase.register(command)
   ├─ validate(slug not reserved, slug not taken)
   ├─ load active required TermsVersions (PLATFORM)
   ├─ verify: all required terms present in command.agreedTermsVersionIds
   ├─ Tenant.create(...)             → TenantRepository.save
   ├─ Shop.createDefault(tenantId)    → ShopRepository.save
   ├─ Account.create(tenantId, email, passwordHash)  → AccountRepository.save
   ├─ Membership.createOwner(accountId, tenantId)     → MembershipRepository.save
   └─ TermsAgreement.create(accountId, termsVersionId)[] → TermsAgreementRepository.saveAll
   ↓
:infra:storage (JPA)
   각 Repository 구현체 (TenantRepositoryImpl 등)
```

### 2.3 현행 스키마 분석

본 TDD에서 도입되는 7개 테이블 + 플랫폼 관리자 분리 모델(`system_admin`)은 이전 CLAUDE.md 결정을 따른다.
`system_admin`은 온보딩에 직접 개입하지 않으므로 본 TDD에서는 스키마만 참조한다.

---

## 3. 아키텍처 설계

### 3.1 계층별 책임 분배

| 계층 | 구성 요소 | 책임 | 설계 근거 |
|------|----------|------|----------|
| Domain | `Tenant`, `Shop`, `Account`, `Membership`, `Terms`, `TermsVersion`, `TermsAgreement`, 각 VO (`TenantSlug`, `Email`, `PasswordHash`) | 생성 불변식, 상태 전이 규칙, 값 검증 | 비즈니스 규칙은 프레임워크와 무관해야 함 |
| Application | `TenantOnboardingUseCase`, `RegisterTenantCommand`, `RegisterTenantResult`, 포트: `TenantRepository`/`ShopRepository`/`AccountRepository`/`MembershipRepository`/`TermsRepository`/`TermsAgreementRepository`, `PasswordHasher`, `SlugReservedPolicy` | 가입 흐름 오케스트레이션, 중복 검증, 필수 약관 검증, 트랜잭션 경계 | 여러 애그리거트 조립은 Application 책임. 도메인 통합 애그리거트 지양 (CLAUDE.md 원칙) |
| App (Backoffice) | `TenantOnboardingController`, `RegisterTenantRequest`, `RegisterTenantResponse`, `TenantOnboardingDtoExtension`, 테넌트-리스 화이트리스트 등록 | HTTP 바인딩, 검증 어노테이션, DTO↔Command 변환 | API 계층은 Spring Web 경계만 |
| Infrastructure | `TenantEntity`/`TenantRepositoryImpl`, `ShopEntity`/`ShopRepositoryImpl`, `AccountEntity`/`AccountRepositoryImpl`, `MembershipEntity`/`MembershipRepositoryImpl`, `TermsEntity`/`TermsVersionEntity`/`TermsRepositoryImpl`, `TermsAgreementEntity`/`TermsAgreementRepositoryImpl`, `BcryptPasswordHasher` | JPA 영속, 도메인↔엔티티 변환 | 도메인 밖으로 JPA 누출 방지 |

### 3.2 처리 흐름 (Command: 테넌트 가입)

```
1. Backoffice Controller 수신
   POST /api/v1/tenants  (X-Tenant-ID 불필요, 인터셉터 화이트리스트)
   Body: { slug, tenantName, shopName, ownerEmail, password, agreedTermsVersionIds[] }

2. RegisterTenantRequest.toCommand() → RegisterTenantCommand

3. TenantOnboardingUseCase.register(command)  [@Transactional]
   3-1. SlugReservedPolicy.assertAvailable(slug)     ← 예약어/포맷 검증
   3-2. TenantRepository.existsBySlug(slug)          ← 중복 검증 (WITHDRAWN 포함 전역 영구 유일)
   3-3. AccountRepository.existsByTenantIdAndEmail(slug, email)  ← (tenantId, email) 복합 유일
   3-4. TermsRepository.findActiveRequiredVersions()  ← PLATFORM 데이터, 활성+필수
        ├─ command.agreedTermsVersionIds ⊇ required.ids 검증 (누락 시 실패)
        └─ (선택 약관 거부는 기록 없음 — agreed=true만 저장)
   3-5. PasswordHasher.hash(password) → PasswordHash VO
   3-6. Tenant.create(slug, name, description) → status=ACTIVE
        TenantRepository.save(tenant)
   3-7. Shop.createDefault(tenantId=slug, name, description, businessRegNo, owner*)
        → status=DRAFT, isDefault=true
        ShopRepository.save(shop)
   3-8. Account.create(tenantId=slug, email, passwordHash) → deleted=false
        AccountRepository.save(account)
   3-9. Membership.createOwner(accountId, tenantId=slug, targetType=TENANT, targetId=slug)
        → role=TENANT_OWNER
        MembershipRepository.save(membership)
   3-10. command.agreedTermsVersionIds.map { TermsAgreement.create(accountId, it, now) }
         TermsAgreementRepository.saveAll(agreements)

4. RegisterTenantResult 반환 → toResponse() → BaseResponse

5. 커밋 후 응답
   { data: { tenantId, shopId, accountId } }
```

### 3.3 설계 대안 분석

| 대안 | 장점 | 단점 | 채택 여부 | 사유 |
|------|------|------|----------|------|
| A. Application 유스케이스가 순차 오케스트레이션 (단일 트랜잭션) | 원자성 쉬움. 도메인 계층 순수 유지. 디버깅 단순. | 서비스가 길어질 수 있음 | ✅ 채택 | CLAUDE.md "애그리거트 조립은 Application 책임" 원칙과 부합 |
| B. 도메인에 `TenantRegistration` 통합 애그리거트 도입 | 규칙을 도메인에 집중 | 서로 다른 영속 단위를 한 애그리거트에 묶어 실질적 애그리거트 경계 붕괴. CLAUDE.md 원칙 위반 | ❌ 기각 | 규칙을 어김 |
| C. 이벤트 기반 Saga로 각 단계 비동기 처리 | 확장성 | 온보딩은 외부 의존이 없고 즉시 완료되어야 하므로 과설계. 보상 트랜잭션 복잡도 증가 | ❌ 기각 | 범위 대비 오버엔지니어링 |
| D. 엔드포인트 위치: `:app:backoffice` | 가입자는 "미래의 테넌트 운영자". backoffice의 사전 인증 구간(가입·로그인·비밀번호 재설정)에 자연스럽게 소속. 가입 완료 후 이어지는 운영자 여정이 동일 모듈에 존재 | backoffice에 사전 인증 화이트리스트 규칙을 도입해야 함 | ✅ 채택 | 액터(역할)가 모듈 분리 기준. 온보딩 신청자는 쇼핑객이 아닌 운영자 |
| E. 엔드포인트 위치: `:app:storefront` | 비인증 공개 엔드포인트 성격만 보면 부합 | storefront는 본질적으로 테넌트 스코프(서브도메인/`X-Tenant-ID`). 온보딩만 예외가 되면 모듈 정체성·인터셉터 정책 훼손. 액터(쇼핑객) 불일치 | ❌ 기각 | "비인증=storefront"는 인증 상태 기준일 뿐 액터 기준이 아님 |

---

## 4. 도메인 모델 설계

### 4.1 애그리거트 경계

각 개념을 **독립 애그리거트**로 정의한다. 이유: 각각 생명주기와 변경 빈도가 다르고, 외래키 강결합을 피하면 멀티테넌트 격리와 성능(Row-level) 모두에 유리.

- `Tenant` (루트: Tenant)
- `Shop` (루트: Shop)
- `Account` (루트: Account)
- `Membership` (루트: Membership) — Account와 별도. 한 계정이 여러 멤버십.
- `Terms` (루트: Terms, 포함: `TermsVersion` 엔티티 N개) — Terms가 버전 생명주기를 관리.
- `TermsAgreement` (루트: TermsAgreement, 불변 기록)

### 4.2 도메인 모델 상세

#### Tenant
- **역할**: 테넌트(쇼핑몰 운영 주체) 식별 및 상태 관리.
- **불변식**:
  - `slug`는 `TenantSlug` VO (소문자 영숫자+하이픈, 3~30자, 예약어 금지).
  - `name`은 1~50자.
  - 생성 시 `status=ACTIVE`.
- **주요 행위**: `create()`, `suspend()`, `withdraw()`.
- **상태 전이**: `ACTIVE → SUSPENDED → ACTIVE`, `ACTIVE|SUSPENDED → WITHDRAWN` (종단).

```
 create()          suspend()/resume()          withdraw()
   →   [ACTIVE]   ⇄   [SUSPENDED]   ──────────→   [WITHDRAWN]
```

> 참고: CLAUDE.md 결정상 가입 시 초기 상태는 `ACTIVE` (이메일 인증 미도입).
> `PENDING`은 상태 머신에 존재하지만 현 단계에서는 진입 경로 없음.

#### Shop
- **역할**: 테넌트가 소유한 개별 쇼핑몰.
- **불변식**:
  - `tenantId`는 필수. 테넌트 1:N.
  - `businessRegistrationNumber`는 포맷 검증만 (정규식 `\d{3}-\d{2}-\d{5}` 등).
  - `ownerEmail`은 `Email` VO.
  - 테넌트당 `isDefault=true` 샵은 정확히 1 (Repository 수준 제약).
- **주요 행위**: `createDefault()`, `activate()`, `suspend()`, `close()`, `promoteToDefault()`.
- **상태 전이**: `DRAFT → ACTIVE → SUSPENDED → ACTIVE`, `ACTIVE|SUSPENDED → CLOSED`.

#### Account
- **역할**: 테넌트 소속 로그인 계정.
- **불변식**:
  - `(tenantId, email)` 복합 유일.
  - 비밀번호는 `PasswordHash` VO (평문 보관 금지).
  - Soft delete: `deleted=false` 기본.
- **주요 행위**: `create()`, `changePassword()`, `softDelete()`.

#### Membership
- **역할**: "누가 어디서 무엇인가" 표현.
- **구조**: `(accountId, tenantId, targetType, targetId, role)` 복합 유일.
- **불변식**:
  - `targetType ∈ {TENANT, SHOP}`.
  - `role ∈ {TENANT_OWNER, TENANT_STAFF, SHOP_CUSTOMER, ...}` (초기엔 TENANT_OWNER, SHOP_CUSTOMER만).
  - 테넌트당 `TENANT_OWNER` 멤버십 정확히 1개 (Repository 수준 제약).
  - 마지막 오너의 비활성/삭제 금지 (도메인에서 거부).

#### Terms / TermsVersion
- **역할**: 약관 정의 + 버전.
- **불변식**:
  - `Terms`는 `tenantId=PLATFORM`.
  - `Terms.required`는 필수/선택 플래그.
  - `Terms`당 활성 `TermsVersion` 최대 1개.
  - `TermsVersion`은 발행 후 본문 변경 불가(immutable).
  - 새 버전 발행 시 이전 활성 버전은 `archived`로 자동 전이.
- **주요 행위**: `Terms.publishNewVersion(content, publisherId)`, `TermsVersion.archive()`.

#### TermsAgreement
- **역할**: "누가 언제 어떤 버전에 동의했는가" 불변 기록.
- **불변식**:
  - `agreed=true`인 경우만 저장 (거부는 기록 없음).
  - 생성 후 내용 변경 불가.
  - `tenantId`는 동의한 계정의 tenantId (테넌트 데이터지만 참조하는 `TermsVersion`은 플랫폼 데이터).

### 4.3 데이터 스키마 설계

```sql
-- 테넌트
CREATE TABLE tenant (
    id            VARCHAR(30)   PRIMARY KEY,  -- slug를 PK로 사용 (tenantId=slug)
    tenant_id     VARCHAR(30)   NOT NULL,     -- 자기 자신 (row-level 필터 일관성)
    name          VARCHAR(50)   NOT NULL,
    description   VARCHAR(500),
    status        VARCHAR(20)   NOT NULL,     -- PENDING|ACTIVE|SUSPENDED|WITHDRAWN
    created_at    DATETIME(6)   NOT NULL,
    updated_at    DATETIME(6)   NOT NULL,
    UNIQUE KEY uk_tenant_id (tenant_id)
);

-- 샵
CREATE TABLE shop (
    id             BIGINT        PRIMARY KEY AUTO_INCREMENT,
    tenant_id      VARCHAR(30)   NOT NULL,
    name           VARCHAR(100)  NOT NULL,
    description    VARCHAR(500),
    status         VARCHAR(20)   NOT NULL,    -- DRAFT|ACTIVE|SUSPENDED|CLOSED
    is_default     BOOLEAN       NOT NULL DEFAULT FALSE,
    business_reg_no VARCHAR(20),
    owner_name     VARCHAR(50),
    owner_phone    VARCHAR(30),
    owner_email    VARCHAR(255),
    created_at     DATETIME(6)   NOT NULL,
    updated_at     DATETIME(6)   NOT NULL,
    KEY idx_shop_tenant (tenant_id),
    UNIQUE KEY uk_shop_tenant_default (tenant_id, is_default)  -- 부분 유일: is_default=TRUE는 최대 1
                                                                -- (MySQL 기본: 기능적 대용으로 앱 레벨 검증 병행)
);

-- 계정
CREATE TABLE account (
    id          BIGINT        PRIMARY KEY AUTO_INCREMENT,
    tenant_id   VARCHAR(30)   NOT NULL,
    email       VARCHAR(255)  NOT NULL,
    password    VARCHAR(100)  NOT NULL,       -- bcrypt hash
    deleted     BOOLEAN       NOT NULL DEFAULT FALSE,
    created_at  DATETIME(6)   NOT NULL,
    updated_at  DATETIME(6)   NOT NULL,
    UNIQUE KEY uk_account_tenant_email (tenant_id, email)
);

-- 멤버십
CREATE TABLE membership (
    id           BIGINT        PRIMARY KEY AUTO_INCREMENT,
    tenant_id    VARCHAR(30)   NOT NULL,
    account_id   BIGINT        NOT NULL,
    target_type  VARCHAR(20)   NOT NULL,      -- TENANT|SHOP
    target_id    VARCHAR(30)   NOT NULL,      -- tenantId or shopId (문자열 통일)
    role         VARCHAR(30)   NOT NULL,      -- TENANT_OWNER|SHOP_CUSTOMER|...
    created_at   DATETIME(6)   NOT NULL,
    updated_at   DATETIME(6)   NOT NULL,
    UNIQUE KEY uk_membership_composite (account_id, tenant_id, target_type, target_id, role),
    KEY idx_membership_tenant (tenant_id)
);

-- 약관
CREATE TABLE terms (
    id          BIGINT        PRIMARY KEY AUTO_INCREMENT,
    tenant_id   VARCHAR(30)   NOT NULL DEFAULT 'PLATFORM',
    name        VARCHAR(100)  NOT NULL,
    description VARCHAR(500),
    required    BOOLEAN       NOT NULL,
    archived    BOOLEAN       NOT NULL DEFAULT FALSE,
    created_at  DATETIME(6)   NOT NULL,
    updated_at  DATETIME(6)   NOT NULL
);

CREATE TABLE terms_version (
    id               BIGINT        PRIMARY KEY AUTO_INCREMENT,
    tenant_id        VARCHAR(30)   NOT NULL DEFAULT 'PLATFORM',
    terms_id         BIGINT        NOT NULL,
    version          INT           NOT NULL,
    content          MEDIUMTEXT    NOT NULL,
    status           VARCHAR(20)   NOT NULL,   -- ACTIVE|ARCHIVED
    effective_from   DATETIME(6)   NOT NULL,
    published_by     BIGINT        NOT NULL,   -- SystemAdmin.id
    created_at       DATETIME(6)   NOT NULL,
    UNIQUE KEY uk_terms_version (terms_id, version),
    KEY idx_terms_version_active (terms_id, status)
);

-- 약관 동의 (불변)
CREATE TABLE terms_agreement (
    id                BIGINT       PRIMARY KEY AUTO_INCREMENT,
    tenant_id         VARCHAR(30)  NOT NULL,
    account_id        BIGINT       NOT NULL,
    terms_version_id  BIGINT       NOT NULL,
    agreed_at         DATETIME(6)  NOT NULL,
    UNIQUE KEY uk_agreement (account_id, terms_version_id),
    KEY idx_agreement_tenant (tenant_id)
);
```

### 4.4 데이터 변환 흐름

```
HTTP JSON
  ↕  (RegisterTenantRequest  ⇄  RegisterTenantResponse)
:app:backoffice DTO
  ↕  (TenantOnboardingDtoExtension: toCommand() / toResponse())
:core:application (RegisterTenantCommand / RegisterTenantResult)
  ↕  (도메인 팩토리: Tenant.create, Shop.createDefault, ...)
:core:domain (Tenant / Shop / Account / Membership / TermsAgreement)
  ↕  (TenantExtension.toEntity() / toDomain() 등 infra 확장함수)
:infra:storage (TenantEntity / ShopEntity / ...)
  ↕  JPA
DB
```

---

## 5. 트랜잭션 설계

### 5.1 트랜잭션 경계

| 연산 | 시작점 | 범위 | 격리 수준 | 사유 |
|------|--------|------|----------|------|
| 테넌트 가입 | `TenantOnboardingUseCase.register()` (@Transactional) | Tenant + Shop + Account + Membership + TermsAgreement(N) | `READ_COMMITTED` (MySQL 기본) | 생성 5종이 하나라도 실패하면 전부 롤백. 외부 연동 없어 긴 트랜잭션 아님. |
| 플랫폼 약관 조회 | `TermsRepository.findActiveRequiredVersions()` (@Transactional(readOnly=true)) | Terms + TermsVersion | `READ_COMMITTED` | 트랜잭션 내부 초반에 호출. 동일 커넥션에서 단순 조회. |

### 5.2 정합성 보장 전략

- **강한 일관성**: 가입은 단일 DB 트랜잭션 → ACID 보장. 이벤트/메시지 큐 미사용.
- **스냅샷 참조**: `TermsAgreement.termsVersionId`는 동의 시점의 구체 버전을 가리키며, 이후 관리자가 버전을 바꿔도 기존 동의 기록은 변함 없음.

### 5.3 이벤트 처리

본 TDD 범위에서는 **이벤트 미발행**. 추후 온보딩 웰컴메일·초기 데이터 시드 등이 추가될 때 `AFTER_COMMIT` 도메인 이벤트를 도입할 지점으로 예약만 해둔다.

---

## 6. 예외 및 실패 처리

### 6.1 예외 분류

모든 예외는 `CoreException(ErrorCode)`를 throw. 새 ErrorCode enum `OnboardingErrorCode`를 `:core:domain`의 하위 enum으로 추가한다.

| 예외 유형 | ErrorCode | 발생 조건 | CoreErrorType → HttpStatus | 사용자 메시지 |
|----------|----------|----------|--------------------------|-------------|
| 예약 slug | `TENANT_SLUG_RESERVED` | slug가 예약어(`admin`, `platform`, `PLATFORM` 등) | VALIDATION → 400 | "사용할 수 없는 식별자입니다." |
| slug 포맷 오류 | `TENANT_SLUG_INVALID_FORMAT` | 정규식 불일치/길이 초과 | VALIDATION → 400 | "식별자 형식이 올바르지 않습니다." |
| slug 중복 | `TENANT_SLUG_DUPLICATED` | 이미 존재(또는 WITHDRAWN) | CONFLICT → 409 | "이미 사용 중인 식별자입니다." |
| 이메일 중복 | `ACCOUNT_EMAIL_DUPLICATED` | (tenantId, email) 충돌 | CONFLICT → 409 | "이미 가입된 이메일입니다." |
| 필수 약관 누락 | `TERMS_REQUIRED_NOT_AGREED` | 활성 필수 약관 중 미동의 존재 | VALIDATION → 400 | "필수 약관에 동의해야 합니다." |
| 약관 버전 무효 | `TERMS_VERSION_INVALID` | 요청한 `agreedTermsVersionIds`에 비활성/미존재 ID 포함 | VALIDATION → 400 | "유효하지 않은 약관입니다." |
| 비밀번호 정책 위반 | `ACCOUNT_PASSWORD_POLICY_VIOLATED` | 최소 8자/영문+숫자 미충족 | VALIDATION → 400 | "비밀번호 정책을 확인해 주세요." |
| 사업자등록번호 포맷 오류 | `SHOP_BUSINESS_REG_NO_INVALID` | 정규식 불일치 | VALIDATION → 400 | "사업자등록번호 형식이 올바르지 않습니다." |

### 6.2 실패 시나리오 및 복구 전략

| # | 시나리오 | 발생 가능성 | 영향 범위 | 복구 전략 |
|---|---------|-----------|----------|----------|
| 1 | slug 중복 (동시 요청 2건) | 중간 | 후속 요청 실패 | DB 유일 제약으로 확정 보호. 두 번째 요청은 `ConstraintViolation`을 `TENANT_SLUG_DUPLICATED`로 변환. |
| 2 | 이메일 중복 | 중간 | 가입 실패 | 사전 조회 + DB 유일 제약 이중 방어. 트랜잭션 롤백. |
| 3 | DB 커넥션 끊김 / 타임아웃 | 낮음 | 가입 중단 | `@Transactional` 전체 롤백. 클라이언트 재시도 안전(멱등 토큰 없으면 중복 slug로 실패). |
| 4 | 약관 버전 관리자 변경과의 경쟁 | 낮음 | 무효 버전 동의 | 트랜잭션 내 `findActiveRequiredVersions()` 기준으로 교차검증. 요청 ID가 현재 활성 목록에 없으면 거부. |
| 5 | `TermsAgreement.saveAll` 도중 제약 위반 | 매우 낮음 | 부분 저장 가능성 | 동일 트랜잭션이므로 전체 롤백. |
| 6 | 비밀번호 해시 실패(라이브러리 장애) | 매우 낮음 | 가입 실패 | `CoreException(INTERNAL_ERROR)`로 500 반환, 로그에 stacktrace. |
| 7 | 남용성 반복 요청 (봇) | 중간 | DB 부하 / slug 고갈 | 본 TDD 범위 외. 인프라 레이어(레이트리밋/CAPTCHA)에서 처리. |

### 6.3 멱등성 보장

본 연산은 **자연적 멱등**:
- 동일 요청 재시도 시 slug/email 유일 제약에 의해 두 번째는 반드시 실패.
- 단, 클라이언트는 첫 번째 요청의 커밋 여부를 알 수 없으므로 **클라이언트측 요청 ID**(X-Idempotency-Key 등)는 후속 개선 항목으로 열어둔다.

---

## 7. 동시성 및 성능

### 7.1 동시성 제어

| 경합 지점 | 제어 방식 | 구현 방법 | 사유 |
|----------|----------|----------|------|
| 동일 slug 동시 가입 | DB 유일 제약 | `tenant.tenant_id` UNIQUE | 슬러그 획득은 극히 드문 경합. 분산 락보다 제약 위반 후 예외 변환이 단순하고 정확. |
| 동일 이메일 동시 가입 | DB 유일 제약 | `account (tenant_id, email)` UNIQUE | 위와 동일. |
| 기본 샵 `is_default=TRUE` 중복 | DB 유일 제약 + 앱 레벨 체크 | MySQL은 NULL 허용 부분 유일 제약이 없어 `(tenant_id, is_default)` 조합 유일 활용. `false` 중복 방지를 위해 `false`는 `NULL` 저장 또는 앱에서 단일 `TRUE` 강제. | 운영 초기 테넌트당 샵 수가 적어 앱 체크로 충분. |
| 약관 활성 버전 변경과 가입 동시 | 트랜잭션 내 재조회 | 가입 트랜잭션이 `findActiveRequiredVersions()`로 재검증 | 관리자 약관 발행은 드문 이벤트. 분산 락 과설계. |

### 7.2 성능 고려사항

| 항목 | 우려 사항 | 대응 전략 | 측정 기준 |
|------|----------|----------|----------|
| 약관 조회 | 매 가입마다 활성 약관 전체 로드 | 약관 수 < 20개로 예상, 단일 쿼리. 필요 시 애플리케이션 캐시(`@Cacheable` + admin 변경 시 evict). | 쿼리 1회, 수십 row |
| 가입 쿼리 수 | 5~N회 INSERT | 단일 트랜잭션 내 배치 없이 순차 실행. 트래픽 낮아 수용 가능. | 평균 6~10 INSERT |
| 인덱스 | `tenant_id` 필터가 전 테이블 공통 | 모든 테넌트 데이터 테이블에 `tenant_id` 인덱스 | N/A |

### 7.3 확장 가능성

**열어둔 확장 포인트**
- `Role`을 enum 값으로 관리하되, `Membership` 테이블 구조는 N:M 확장을 허용.
- 온보딩 후처리(웰컴메일, 기본 카테고리 시드 등)는 도메인 이벤트 발행 지점(`TenantRegisteredEvent`)을 `AFTER_COMMIT`으로 추가 가능.
- `PENDING` 상태를 이용한 이메일 인증 흐름은 상태 머신에 이미 정의되어 있어 인증 유스케이스만 추가하면 됨.
- 테넌트-리스 엔드포인트 화이트리스트 구조는 `X-Idempotency-Key` 기반 멱등 토큰 확장과 호환.

**의도적으로 닫아둔 제약**
- 오너 1명 고정 — 이관/복수 오너 기능 추가 시 `Membership` 유일 제약 완화 + 도메인 규칙 변경 필요.
- slug 불변 — 변경 요구 발생 시 별도 "테넌트 재발급" 유스케이스로 다룬다.
- WITHDRAWN slug 재사용 금지 — 정책 변경 시 유일 제약을 `(slug, withdrawn_at IS NULL)` 부분 인덱스 등으로 전환 필요.

---

## 8. 변경 파일 목록

| # | 파일 | 모듈 | 변경 유형 | 설명 |
|---|------|------|----------|------|
| 1 | `com/wooyong/demo/core/domain/tenant/Tenant.kt` | :core:domain | 신규 | Tenant 루트 + 팩토리 |
| 2 | `com/wooyong/demo/core/domain/tenant/TenantSlug.kt` | :core:domain | 신규 | Slug VO |
| 3 | `com/wooyong/demo/core/domain/tenant/TenantStatus.kt` | :core:domain | 신규 | 상태 enum |
| 4 | `com/wooyong/demo/core/domain/shop/Shop.kt` + `ShopStatus.kt` | :core:domain | 신규 | |
| 5 | `com/wooyong/demo/core/domain/account/Account.kt` + `Email.kt` + `PasswordHash.kt` | :core:domain | 신규 | |
| 6 | `com/wooyong/demo/core/domain/membership/Membership.kt` + `MembershipRole.kt` + `MembershipTargetType.kt` | :core:domain | 신규 | |
| 7 | `com/wooyong/demo/core/domain/terms/Terms.kt` + `TermsVersion.kt` + `TermsAgreement.kt` | :core:domain | 신규 | |
| 8 | `com/wooyong/demo/core/domain/onboarding/OnboardingErrorCode.kt` | :core:domain | 신규 | ErrorCode enum |
| 9 | `com/wooyong/demo/core/application/onboarding/TenantOnboardingUseCase.kt` | :core:application | 신규 | 유스케이스 |
| 10 | `com/wooyong/demo/core/application/onboarding/RegisterTenantCommand.kt` + `RegisterTenantResult.kt` | :core:application | 신규 | |
| 11 | `com/wooyong/demo/core/application/onboarding/SlugReservedPolicy.kt` | :core:application | 신규 | 예약어 정책 (도메인으로 올려도 됨, 현 설계는 application에 보관) |
| 12 | `com/wooyong/demo/core/application/.../*Repository.kt`, `PasswordHasher.kt` | :core:application | 신규 | 포트 인터페이스 |
| 13 | `com/wooyong/demo/storage/tenant/*`, `shop/*`, `account/*`, `membership/*`, `terms/*`, `termsagreement/*` | :infra:storage | 신규 | Entity + Repository 구현 + Extension |
| 14 | `com/wooyong/demo/storage/password/BcryptPasswordHasher.kt` | :infra:storage or :infra:internal | 신규 | 해시 구현 |
| 15 | `com/wooyong/demo/backoffice/onboarding/TenantOnboardingController.kt` | :app:backoffice | 신규 | 공개 엔드포인트 |
| 16 | `com/wooyong/demo/backoffice/onboarding/TenantOnboardingDtoExtension.kt` | :app:backoffice | 신규 | |
| 17 | `com/wooyong/demo/backoffice/onboarding/dto/TenantOnboardingRequests.kt` + `Responses.kt` | :app:backoffice | 신규 | |
| 18 | `com/wooyong/demo/backoffice/config/TenantInterceptor.kt` | :app:backoffice | 신규 또는 수정 | `/api/v1/tenants`, `/api/v1/public/terms/**` 화이트리스트 |
| 19 | `sql/tenant/*.sql`, `sql/shop/*.sql`, `sql/account/*.sql`, `sql/membership/*.sql`, `sql/terms/*.sql`, `sql/terms_agreement/*.sql` | 루트 | 신규 DDL | DDL-First |

---

## 9. 검증 계획

| 시나리오 | 유형 | 검증 내용 | 예상 결과 |
|----------|------|----------|----------|
| 정상 가입 | 통합 테스트 | 모든 필드 유효 + 필수 약관 전체 동의 | 200 OK, 5개 테이블 row 생성, `is_default=true` 샵 1개 |
| slug 예약어 | 단위 | `admin`, `platform` slug | `TENANT_SLUG_RESERVED` |
| slug 포맷 오류 | 단위 | 대문자, 길이 2 | `TENANT_SLUG_INVALID_FORMAT` |
| slug 중복 (동시) | 통합 | 동일 slug 2건 병렬 제출 | 1건 성공, 1건 `TENANT_SLUG_DUPLICATED` |
| 필수 약관 누락 | 단위 | 필수 2개 중 1개만 동의 | `TERMS_REQUIRED_NOT_AGREED`, 어떤 row도 생성 안 됨 |
| 선택 약관 거부 | 통합 | 필수만 동의 | 성공, 선택 약관 `terms_agreement` row 없음 |
| 비밀번호 정책 | 단위 | 7자 | `ACCOUNT_PASSWORD_POLICY_VIOLATED` |
| 이메일 중복 | 통합 | 동일 tenant+email 2회 | 2번째 `ACCOUNT_EMAIL_DUPLICATED` |
| 트랜잭션 롤백 | 통합 | 마지막 단계(TermsAgreement)에서 강제 예외 발생 | Tenant/Shop/Account/Membership 모두 롤백 |
| 테넌트-리스 호출 | 통합 | `X-Tenant-ID` 미지정 | 200 OK (화이트리스트 동작) |
| 기본 샵 2개 방어 | 통합 | `is_default=true`로 2번째 샵 저장 시도 | 제약 위반 예외 |
| 마지막 오너 삭제 금지 | 단위 | 유일 오너 멤버십 삭제 시도 | 도메인 예외 |

---

## 부록: 요청/응답 예시

**Request**
```http
POST /api/v1/tenants HTTP/1.1
Content-Type: application/json

{
  "slug": "acme-shop",
  "tenantName": "ACME",
  "shopName": "ACME Store",
  "businessRegistrationNumber": "123-45-67890",
  "ownerName": "홍길동",
  "ownerPhone": "010-1234-5678",
  "ownerEmail": "owner@acme.example",
  "password": "Secret123",
  "agreedTermsVersionIds": [11, 12]
}
```

**Response (성공)**
```json
{
  "data": {
    "tenantId": "acme-shop",
    "shopId": 101,
    "accountId": 2001
  }
}
```

**Response (실패 — slug 중복)**
```json
{
  "error": {
    "code": "TENANT_SLUG_DUPLICATED",
    "message": "이미 사용 중인 식별자입니다."
  }
}
```
