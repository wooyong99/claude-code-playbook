# External 계층 ApiClient 구현 전략

[external-layer-guidelines.md](../external-layer-guidelines.md)의 보편 원칙(R1–R4) 위에서,
이 프로젝트가 선택한 External 계층 ApiClient 세부 구현 전략을 정의한다.

---

## 이 프로젝트의 전략

> **[커스터마이징 영역]** 새 프로젝트 적용 시 이 섹션을 아래 템플릿으로 교체한다.

**HTTP 클라이언트**: {RestClient / RestTemplate / WebClient 중 선택}

**로깅 방식**: {AOP 어노테이션 / 수동 로깅 / 인터셉터 중 선택}

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| HTTP 클라이언트 구성 | `{Provider}Config` | [api-client-http-client.md](api-client-http-client.md) |
| 외부 API 호출 로깅 | `{Provider}ApiClient` | [api-client-logging.md](api-client-logging.md) |

**Post-Work Verification 체크리스트**:
- [../adapter-convention.md](../adapter-convention.md)
- [../api-client-convention.md](../api-client-convention.md)
- [api-client-http-client.md](api-client-http-client.md)
- [api-client-logging.md](api-client-logging.md)
- [../dto-convention.md](../dto-convention.md)
- [../exception-convention.md](../exception-convention.md)
- [../errorcode-convention.md](../errorcode-convention.md)
- [../config-convention.md](../config-convention.md)
- [../mock-adapter-convention.md](../mock-adapter-convention.md)

---

## 역할 정의

ApiClient 구현 시 관여하는 역할 정의.

| 역할 | 설명 | 구현 컴포넌트 |
|-----|------|------------|
| HTTP 호출 | 외부 엔드포인트 요청·응답·예외 변환 | `{Provider}ApiClient` |
| 토큰 캐싱 | Bearer Token 발급·갱신·만료 관리 | `{Provider}TokenHolder` |
| HTTP 클라이언트 빈 | Provider 전용 HTTP 클라이언트 생성·구성 | `{Provider}Config` |

---

## 전략 정의 템플릿

새 프로젝트가 "이 프로젝트의 전략" 섹션을 채울 때 사용하는 템플릿:

```markdown
**HTTP 클라이언트**: {선택한 클라이언트}

**로깅 방식**: {선택한 로깅 방식}

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| HTTP 클라이언트 구성 | `{Provider}Config` | [api-client-http-client.md](api-client-http-client.md) |
| 외부 API 호출 로깅 | `{Provider}ApiClient` | [api-client-logging.md](api-client-logging.md) |

**Post-Work Verification 체크리스트**:
- [../adapter-convention.md](../adapter-convention.md)
- [../api-client-convention.md](../api-client-convention.md)
- [api-client-http-client.md](api-client-http-client.md)
- [api-client-logging.md](api-client-logging.md)
- [../dto-convention.md](../dto-convention.md)
- [../exception-convention.md](../exception-convention.md)
- [../errorcode-convention.md](../errorcode-convention.md)
- [../config-convention.md](../config-convention.md)
- [../mock-adapter-convention.md](../mock-adapter-convention.md)
```
