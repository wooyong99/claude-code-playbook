# claude-code-playbook

AI 에이전트(Claude Code)를 활용해 **대규모 서비스를 안정적으로 구축·유지보수**하기 위한 실전 템플릿.
엄격한 아키텍처 문서와 AI 가드레일을 한 세트로 묶어, 어떤 프로젝트에도 이식해 쓸 수 있도록 설계되었다.

> 예시 도메인: **e-커머스 플랫폼** (Kotlin + Spring Boot + React).
> 본 리포지토리는 *실행 코드가 없는 문서·설정 템플릿*이며, 코드 베이스를 새로 시작할 때 이 구조를 베이스로 삼는 것을 전제로 한다.

---

## 이 리포지토리가 해결하려는 문제

AI 에이전트에게 기능 구현을 맡길 때 가장 큰 리스크는 두 가지다.

1. **예측 불가능한 이탈** — 에이전트가 허용되지 않은 파일을 수정하거나, 아키텍처 규칙을 자기 판단으로 어기거나, 잘못된 방향으로 리팩토링을 진행한다.
2. **암묵지의 부재** — 사람 사이에는 "이 프로젝트는 이렇게 짠다"라는 암묵적 합의가 있지만, 에이전트는 매 세션마다 제로에서 시작한다.

이 리포지토리는 두 축으로 접근한다.

- **`docs/`** — 프로젝트의 모든 규칙·제약을 **명시된 문서**로 고정. 에이전트가 추측하지 않고 문서를 참조하게 만든다.
- **`.claude/`** — 에이전트의 행동을 제한·유도하는 **가드레일**(서브에이전트, 커스텀 커맨드, 훅, 권한). 에이전트가 스스로 검증-수정 루프를 돌도록 오케스트레이션한다.

---

## 핵심 아이디어 — 3줄 요약

| 축 | 수단 | 효과 |
|----|------|-----|
| 규칙을 명시한다 | `docs/` (아키텍처·컨벤션·정책) | 에이전트가 "어떻게 짜야 하는가"를 추측하지 않는다 |
| 역할을 분리한다 | `code-writer` ↔ `architecture-reviewer` 서브에이전트 | 작성자 편향을 제3자가 교정한다 |
| 루프를 돌린다 | `implement` 스킬 | 작성 → 검토 → 수정을 PASS까지 자동 수렴 |

---

## 디렉토리

```
claude-code-playbook/
├── .claude/              # AI 에이전트 가드레일 (서브에이전트, 커맨드, 훅, 권한)
│   ├── CLAUDE.md         # 프로젝트 컨텍스트 (에이전트가 항상 로드)
│   ├── agents/           # 역할별 서브에이전트 정의
│   ├── skills/           # 스킬 (오케스트레이션·자동화 워크플로)
│   └── settings.json     # 훅, 권한 등 운용 설정
│
├── docs/                 # 프로젝트 규칙·설계 문서 (에이전트와 사람이 공유하는 Source of Truth)
│   ├── backend/          # 백엔드 아키텍처·컨벤션·정책
│   └── frontend/         # 프론트엔드 아키텍처·컨벤션·성능·UI/UX
│
└── README.md             # 현재 문서 (목차)
```

세부는 아래 문서에서 이어진다.

---

## 문서 목차

| 단계 | 문서 | 언제 보나 |
|-----|------|----------|
| 1. 철학 이해 | [PHILOSOPHY.md](PHILOSOPHY.md) | "왜 이 구조인가?"가 궁금할 때 |
| 2. 시작하기 | [GETTING_STARTED.md](GETTING_STARTED.md) | 클론 직후, 프로젝트 시작 시 |
| 3. 문서 탐색 | [DOCS_GUIDE.md](DOCS_GUIDE.md) | `docs/`에서 규칙을 찾을 때 |
| 4. 가드레일 | [CLAUDE_SETUP.md](CLAUDE_SETUP.md) | `.claude/` 구조와 커스터마이징 |
| 5. 운용 흐름 | [WORKFLOW.md](WORKFLOW.md) | `implement` 스킬로 기능을 구현할 때 |

---

## 빠른 시작 (30초)

```bash
# 1. 클론
git clone <this-repo> my-project
cd my-project

# 2. Claude Code 실행 (이 리포지토리 루트에서)
claude

# 3. 예시 — 요구사항을 마일스톤 단위로 분해·구현 (implement 스킬이 자동으로 트리거됨)
"카테고리 CRUD를 추가해줘"
```

Claude Code가 아직 설치되어 있지 않다면 [GETTING_STARTED.md](GETTING_STARTED.md) 참조.

---

## 이 템플릿을 내 프로젝트에 맞게 고치려면

- **프로젝트 컨텍스트를 설정할 때** → `setup-project-context` 스킬 실행. `.claude/CLAUDE.md`의 플레이스홀더를 인터뷰 방식으로 채운다.
- **도메인을 바꿀 때** → `docs/backend/` · `docs/frontend/` 하위 문서의 도메인 예시와 모듈 이름을 갈아끼운다. 구조는 그대로 두고 내용만 바꾸는 것을 권장.
- **규칙을 추가할 때** → `docs/backend/architecture/*` 또는 `docs/backend/policies/*`에 새 문서를 추가하고, `architecture-reviewer.md`의 **Source of Truth** 목록에 등록한다.
- **에이전트 역할을 늘릴 때** → `.claude/agents/`에 새 서브에이전트를 추가하고, `implement` 스킬(또는 새 스킬)에서 호출 지점을 정의한다.

자세한 절차는 [CLAUDE_SETUP.md](CLAUDE_SETUP.md#커스터마이징)에.
