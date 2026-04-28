# Application 계층 구현 전략

[Application Layer Guidelines](../application-layer-guidelines.md)의 보편 원칙(R1–R4) 위에서, 이 프로젝트가 선택한 Application 계층 내부 구현 전략을 정의한다.

새 프로젝트에 이 플레이북을 적용할 때는 **"이 프로젝트의 전략"** 섹션을 교체하고, 각 역할에 맞는 컨벤션 문서를 작성한다.

---

## 이 프로젝트의 전략

> **[커스터마이징 영역]** 새 프로젝트 적용 시 이 섹션을 아래 [템플릿](#전략-정의-템플릿)으로 교체한다.

**전략명**: UseCase + Flow 계층 분리

**흐름**:

```
UseCase → Flow → Validator / Handler / Policy → Port
```

**선택 이유**: 여러 UseCase에서 공통 업무 흐름을 재사용하기 위해 Flow를 별도 계층으로 분리한다.

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| Entry Point | UseCase | [use-case-convention.md](use-case-convention.md) |
| Flow Orchestrator | Flow | [flow-convention.md](flow-convention.md) |
| Rule Checker | Validator | [validator-convention.md](validator-convention.md) |
| ACL / Coordinator | Handler | [handler-convention.md](handler-convention.md) |
| Strategy | Policy | [policy-convention.md](policy-convention.md) |
| Event Handler | EventHandler | [event-handler-convention.md](event-handler-convention.md) |
| Assembler | Mapper | [mapper-convention.md](mapper-convention.md) |

패키지 구조: [package-structure.md](package-structure.md)

**Post-Work Verification 체크리스트**:

- [package-structure.md](package-structure.md)
- [use-case-convention.md](use-case-convention.md)
- [flow-convention.md](flow-convention.md)
- [validator-convention.md](validator-convention.md)
- [handler-convention.md](handler-convention.md)
- [policy-convention.md](policy-convention.md)
- [event-handler-convention.md](event-handler-convention.md)
- [mapper-convention.md](mapper-convention.md)

---

## 역할 정의

어떤 전략을 선택하든 아래 역할 중 필요한 것을 정의한다. 역할이 필요하지 않으면 생략한다.

| 역할 | 설명 | 구현 예시 |
|-----|------|---------|
| Entry Point | 외부 요청 수신, 내부 흐름 조립 | `Service`, `UseCase`, `CommandHandler` |
| Flow Orchestrator | 재사용 가능한 업무 흐름 캡슐화 | `Flow`, `BusinessService`, `Saga` |
| Rule Checker | 조회된 데이터 기반 규칙 검증 — 인프라 의존 없음 | `Validator`, `Guard`, `Specification` |
| ACL / Coordinator | 도메인 간 경계 보호, 공통 로직 재사용 | `Handler`, `Facade`, `ApplicationService` |
| Strategy | 확장 가능한 행위 분기 외부화 | `Policy`, `Strategy`, `Rule` |
| Event Handler | 트랜잭션 커밋 후 부수 효과 처리 | `EventHandler`, `DomainEventHandler` |
| Assembler | 도메인 결과 → 외부 응답 변환 | `Mapper`, `Assembler`, `DtoFactory` |

**복잡도별 추천 구성**:

| 상황 | 포함할 역할 |
|-----|-----------|
| 팀이 작고 도메인이 단순 | Entry Point + Assembler |
| 여러 진입점에서 공통 흐름 재사용 필요 | + Flow Orchestrator |
| 도메인 규칙 검증이 복잡 | + Rule Checker |
| 여러 도메인 간 협력 필요 | + ACL / Coordinator |
| 행위 유형이 N개 이상으로 확장 예정 | + Strategy |
| 커밋 후 알림·연동 처리 필요 | + Event Handler |

---

## 전략 정의 템플릿

```markdown
**전략명**: {전략 이름 — 예: "Service 중심 단순 계층", "CQRS"}

**흐름**:
{컴포넌트 흐름 다이어그램}

**선택 이유**: {이 전략을 선택한 이유}

**역할별 컴포넌트**:

| 역할 | 컴포넌트 | 컨벤션 문서 |
|-----|---------|-----------|
| Entry Point | {컴포넌트명} | {링크 또는 `-`} |
| Assembler   | {컴포넌트명} | {링크 또는 `-`} |
| (필요한 역할만 추가) | | |

**Post-Work Verification 체크리스트**:
- {컨벤션 문서 링크 목록}
```
