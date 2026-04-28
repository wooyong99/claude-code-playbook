# App 계층 구현 전략

[App Layer Guidelines](../app-layer-guidelines.md)의 보편 원칙(R1–R4) 위에서, 이 프로젝트가 선택한 App 계층 인증·보안·컨텍스트 전파 전략을 정의한다.

새 프로젝트에 이 플레이북을 적용할 때는 **"이 프로젝트의 전략"** 섹션을 교체하고, 각 역할에 맞는 컨벤션 문서를 작성한다.

---

## 이 프로젝트의 전략

> **[커스터마이징 영역]** 새 프로젝트 적용 시 이 섹션을 아래 [전략 정의 템플릿](#전략-정의-템플릿)으로 교체한다.

**전략명**: JWT Stateless 인증 + 헤더 기반 멀티테넌트

**흐름**:

```
HTTP Request
  ↓ JwtAuthenticationFilter
[JWT 파싱 → SecurityContext 설정 → TenantContextHolder 주입]
  ↓ MdcLoggingFilter
[requestId, tenantId MDC 주입]
  ↓ Controller
```

**선택 이유**: REST API 서버에서 수평 스케일이 용이하고, 헤더로 테넌트를 식별하면 URL 설계를 테넌트 중립적으로 유지할 수 있다.

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 인증 필터 | `JwtAuthenticationFilter` | - |
| 토큰 제공자 | `JwtTokenProvider` | - |
| 보안 설정 | `SecurityConfig` | - |
| 컨텍스트 홀더 | `TenantContextHolder` | - |
| 컨텍스트 주입기 | `MdcLoggingFilter` | - |

패키지 배치: [common.md](common.md) `security/` · `logging/` 섹션 참고

**Post-Work Verification 체크리스트**:

- [api-convention.md](api-convention.md)
- [rest-design-convention.md](rest-design-convention.md)
- [exception-handling-convention.md](exception-handling-convention.md)
- [file-structure.md](file-structure.md)
- [common.md](common.md)

---

## 역할 정의

어떤 전략을 선택하든 아래 역할 중 필요한 것을 정의한다. 역할이 필요하지 않으면 생략한다.

| 역할 | 설명 | 구현 예시 |
|-----|------|---------|
| 인증 필터 | 요청에서 자격증명을 추출하고 SecurityContext에 인증 주체를 설정 | `JwtAuthenticationFilter`, `SessionAuthFilter`, `ApiKeyAuthFilter` |
| 토큰 제공자 | 토큰 생성·파싱·검증. 인증 필터가 사용 | `JwtTokenProvider`, `JwtDecoder` |
| 보안 설정 | `SecurityFilterChain`, URL·역할별 접근 제어 규칙 정의 | `SecurityConfig`, `MethodSecurityConfig` |
| 컨텍스트 홀더 | 요청 스코프 내 인증 주체·테넌트 정보 보관. 필터 종료 시 반드시 제거 | `TenantContextHolder`, `SecurityContextHolder` |
| 컨텍스트 주입기 | requestId, tenantId, userId를 MDC·ThreadLocal에 주입·제거 | `MdcLoggingFilter`, `TenantContextFilter` |
| 테넌트 리졸버 | 요청에서 tenantId를 추출하는 방법 정의 | `HeaderTenantResolver`, `JwtClaimTenantResolver` |

**인증 전략 옵션**:

| 전략 | 특징 |
|-----|------|
| JWT Stateless | 매 요청마다 토큰 검증. 세션 스토어 불필요. 수평 스케일 용이 |
| Session 기반 | 서버 세션 스토어(Redis 등) 필요. 즉시 무효화 가능 |
| API Key | 서비스 간 인증. 사용자 맥락이 없는 내부 API에 적합 |
| OAuth2 Resource Server | 외부 IdP(Keycloak, Cognito 등) 연동. 토큰 검증을 IdP에 위임 |

**멀티테넌트 전략 옵션**:

| 전략 | tenantId 추출 위치 |
|-----|------------------|
| 헤더 기반 | `X-Tenant-Id` 헤더 |
| JWT Claim 기반 | JWT 페이로드 내 `tenant_id` claim |
| 단일 테넌트 | 멀티테넌시 없음 — 컨텍스트 홀더 불필요 |

---

## 전략 정의 템플릿

```markdown
**전략명**: {전략 이름 — 예: "JWT Stateless + 헤더 멀티테넌트", "Session 기반 단일 테넌트"}

**흐름**:
{필터 체인 흐름 다이어그램}

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| 인증 필터     | {컴포넌트명} | {링크 또는 `-`} |
| 보안 설정     | {컴포넌트명} | {링크 또는 `-`} |
| 컨텍스트 홀더 | {컴포넌트명 또는 `-`} | {링크 또는 `-`} |
| (필요한 역할만 추가) | | |

**Post-Work Verification 체크리스트**:
- [api-convention.md](api-convention.md)
- [rest-design-convention.md](rest-design-convention.md)
- [exception-handling-convention.md](exception-handling-convention.md)
- [file-structure.md](file-structure.md)
- {추가 컨벤션 문서 링크}
```
