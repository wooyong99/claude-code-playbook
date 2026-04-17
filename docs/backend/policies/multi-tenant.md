# Multi-Tenant Strategy

플랫폼의 테넌트 격리 전략과 데이터 분류 기준.

---

## 격리 전략

- **Shared Database, Row-level tenancy** — 단일 DB 내에서 `tenant_id` 컬럼으로 행 단위 격리.
- 모든 **테넌트 스코프** 영속 도메인 모델은 `tenantId: String` 필드를 포함한다.
- 도메인 로직은 테넌트를 인지하지 않는다. 격리는 인프라 책임.
- 테넌트 컨텍스트는 요청 스코프 `ThreadLocal`로 전파한다 (`infra:security`의 `TenantContextHolder`).

---

## 데이터 분류

| 분류 | tenant_id 컬럼 | 예시 |
|------|--------------|------|
| 테넌트 데이터 | 실제 테넌트 slug | Shop, Account, Membership |
| 플랫폼 데이터 | 컬럼 없음 | Terms, TermsVersion, SystemAdmin |
| 동의 스냅샷 | 실제 테넌트 slug | TermsAgreement |
| 레지스트리 | 컬럼 없음 | Tenant |

---

## 접근 규칙

- 플랫폼 데이터 쓰기는 `:app:admin`에서만 수행한다. 다른 모듈은 읽기 전용.
- 테넌트-리스 엔드포인트(온보딩 등)는 `TenantInterceptor` 화이트리스트로 관리한다.

---

## 앱 모듈별 테넌트 식별

| 모듈 | 사용자 | 인증 컨텍스트 | 테넌트 식별 방식 |
|------|--------|-------------|---------------|
| `:app:storefront` | 샵 방문 고객 | Account (SHOP_CUSTOMER) | 서브도메인 / `X-Tenant-ID` 필수 |
| `:app:backoffice` | 테넌트 운영자 | Account (TENANT_OWNER 등) | 인증된 멤버십에서 파생 |
| `:app:admin` | 시스템 관리자 | SystemAdmin | 없음 (테넌트 필터 우회) |
