# GETTING_STARTED — 클론 후 시작하기

이 리포지토리는 **코드가 아닌 문서·설정 템플릿**이다.
클론 후 Claude Code를 이 디렉토리에서 실행하면 가드레일이 자동으로 활성화된다.

---

## 1. 사전 준비

### 필수

- **Claude Code CLI** — [공식 설치 가이드](https://docs.claude.com/en/docs/claude-code) 참조.
  ```bash
  # macOS (Homebrew 예시)
  brew install anthropics/claude/claude-code

  # 또는 npm
  npm install -g @anthropic-ai/claude-code
  ```
- **Git**

### 권장 (도메인 구현 시)

예시 도메인(Kotlin + Spring Boot e-커머스)을 그대로 따라가려면:

- JDK 21 이상
- Node.js 20 이상 (프론트엔드)
- Docker (로컬 DB용)

> 도메인을 바꿀 계획이면 이 도구들은 필요 없다. `docs/` 하위의 Kotlin/Spring 관련 문서를 자신의 스택에 맞게 교체한다.

---

## 2. 클론

```bash
git clone <this-repo> my-project
cd my-project
```

---

## 3. Claude Code 실행

리포지토리 루트에서:

```bash
claude
```

최초 실행 시 Claude Code가 `.claude/CLAUDE.md`와 `.claude/settings.json`을 자동으로 로드한다.
`CLAUDE.md`에 프로젝트 컨텍스트(도메인, 아키텍처 요약, 문서 맵)가 정의되어 있어, 에이전트가 매 세션마다 동일한 전제 아래에서 시작한다.

### 확인

Claude에게 아래와 같이 물어본다:

```
이 프로젝트는 뭐 하는 프로젝트야?
```

`.claude/CLAUDE.md`에 정의된 내용을 기반으로 답변하면 컨텍스트 로드 성공.

---

## 4. 첫 작업 — `implement` 스킬 체험

`implement` 스킬이 자동 트리거되는 기능 구현 요청을 해본다:

```
"테넌트 온보딩 API를 추가해줘"
```

Claude가 다음 순서로 진행한다:

1. 요구사항 재진술 → 사용자 확인
2. 마일스톤 분할 → 사용자 확인
3. 각 마일스톤마다:
   - `design-writer` 에이전트가 기술설계문서(TDD) 작성
   - `code-writer` 에이전트가 TDD 기반으로 구현
   - `architecture-reviewer` 에이전트가 문서 기반 검토
   - 위반 시 자동 수정 → PASS까지 반복
4. 완료 보고

자세한 흐름은 [WORKFLOW.md](WORKFLOW.md) 참조.

---

## 5. 내 프로젝트에 맞게 바꾸기

### CLAUDE.md 프로젝트 컨텍스트 설정

`.claude/CLAUDE.md`는 플레이스홀더(`{프로젝트명}`, `{비즈니스 목표}` 등)로 된 템플릿이다.
`setup-project-context` 스킬로 인터뷰 방식으로 채울 수 있다:

```
/setup-project-context
```

스킬이 현재 코드베이스를 탐색하거나 신규 프로젝트 구조를 제안하고, 비즈니스 목표와 사용자 유형을 물어본 뒤 CLAUDE.md를 완성한다.

### 도메인이 e-커머스가 아니라면

1. `docs/backend/` 하위 모든 문서에서 **예시 도메인 이름**(Product, Category 등)을 자신의 도메인으로 치환.
2. 필요 없는 가이드라인은 삭제, 부족한 것은 추가.

### 기술 스택이 Kotlin/Spring이 아니라면

1. `docs/backend/architecture/` 의 레이어 구조는 Clean Architecture + DDD 베이스이므로 대부분의 스택에 적용 가능.
2. 언어·프레임워크 특화 컨벤션(예: QueryDSL, `@Valid` 등)은 자신의 스택의 동등물로 치환.
3. `.claude/agents/code-writer.md`의 "Mandatory References" 섹션을 수정된 문서 경로로 업데이트.

### 에이전트 역할을 추가하려면

[CLAUDE_SETUP.md — 커스터마이징](CLAUDE_SETUP.md#커스터마이징) 참조.

---

## 6. 자주 쓰는 명령

| 명령 | 설명 |
|------|------|
| `claude` | 세션 시작 |
| `"<요구사항>"` | 요구사항을 마일스톤으로 분해해 구현 (implement 스킬 자동 트리거) |
| `/help` | Claude Code 도움말 |
| `/model` | 사용 모델 변경 |

---

## 다음 단계

- [PHILOSOPHY.md](PHILOSOPHY.md) — 이 구조를 만든 설계 사상
- [DOCS_GUIDE.md](DOCS_GUIDE.md) — `docs/` 문서 맵
- [CLAUDE_SETUP.md](CLAUDE_SETUP.md) — 가드레일 커스터마이징
- [WORKFLOW.md](WORKFLOW.md) — `/implement` 커맨드 상세 흐름
