# CLAUDE_SETUP — `.claude/` 가드레일 구조

`.claude/`는 Claude Code가 세션 시작 시 자동 로드하는 **프로젝트 전용 설정**이다.
이 디렉토리의 파일들이 에이전트의 행동을 제한하고 오케스트레이션한다.

---

## 구조

```
.claude/
├── CLAUDE.md                   # 프로젝트 컨텍스트 (에이전트가 항상 로드)
├── agents/
│   ├── design-writer.md         # 기술설계문서(TDD) 작성 서브에이전트
│   ├── code-writer.md           # 코드 작성 서브에이전트
│   └── architecture-reviewer.md # 아키텍처 검토 서브에이전트
├── skills/
│   └── implement/
│       ├── SKILL.md             # implement 스킬 (D→A→B 오케스트레이션)
│       └── references/          # 에이전트별 입출력 계약 문서
│           ├── design-writer-contract.md
│           ├── code-writer-contract.md
│           └── architecture-reviewer-contract.md
├── settings.json               # 팀 공유 설정 (훅, 권한 등, 커밋 대상)
└── settings.local.json         # 로컬 전용 설정 (커밋 제외)
```

---

## 1. `CLAUDE.md` — 프로젝트 헌장

**역할**: Claude Code가 세션 시작 시 자동으로 로드하는 전역 컨텍스트.
에이전트가 매번 물어봐야 할 "이 프로젝트는 어떤 프로젝트인가?"에 대한 답을 상시 고정한다.

### 담는 내용

- 비즈니스 목표·사용자 유형
- 프로젝트 디렉토리 개요
- **문서 맵** — `docs/` 하위 가이드라인으로 가는 링크 테이블
- 작업 원칙 (규칙은 하위 문서를 참조하라는 메타 지침)
- 코드 작업 오케스트레이션 방식 (`implement` 스킬 소개)

### 유지 원칙

- **규칙의 내용은 여기 쓰지 않는다.** 링크만 둔다. 규칙의 진짜 위치는 `docs/` 하위.
- 길어지지 않게 유지. 상세는 `docs/`로 분리.

---

## 2. `agents/` — 역할별 서브에이전트

메인 Claude(오케스트레이터)가 특정 작업을 **다른 인스턴스에 위임**하는 서브에이전트 정의.

### 왜 분리하는가

- **컨텍스트 격리** — 메인 세션의 맥락이 오염되지 않는다
- **역할 특화** — 각 에이전트에 필요한 문서·도구만 노출
- **편향 제거** — 작성자와 검토자가 서로 다른 인스턴스

### 현재 정의된 에이전트

#### `design-writer.md`

- **역할**: 마일스톤별 기술설계문서(TDD) 작성. 코드 작성 전에 호출되어 설계 방향을 확정한다.
- **도구**: 전체 사용 (Read, Write, Skill 등). 단, 코드 파일 수정은 금지.
- **제약**:
  - **코드 작성 금지** — 설계 문서만 작성
  - **사용자와 직접 대화 금지** — 모든 소통은 오케스트레이터를 통해
  - TDD 필요 여부를 판단해 불필요하면 `TDD_SKIPPED`로 스킵
- **반환 포맷**: `TDD_CREATED: <경로>` / `TDD_SKIPPED: <이유>` / `CONTEXT_CHECKPOINT: ...`

#### `code-writer.md`

- **역할**: Kotlin 코드 작성·수정·테스트. 요구사항과 TDD를 받아 파일을 편집하고 빌드를 통과시킨다.
- **도구**: 전체 사용 (Read, Write, Edit, Bash 등)
- **제약**:
  - **자기검증 금지** — 자기 코드의 아키텍처 준수 여부는 판단하지 않는다 (편향 방지)
  - **사용자와 직접 대화 금지** — 궁금한 점은 요약에 기록 → 메인이 전달
  - **Scope 준수** — 지시되지 않은 파일·리팩토링 금지
- **반환 포맷**: 작업 유형별 고정 포맷 (변경 파일 목록, 설계 결정, 빌드 결과, 불확실한 부분)

#### `architecture-reviewer.md`

- **역할**: `docs/backend/architecture/*` · `docs/backend/policies/*` 준수 여부 검토.
- **도구**: `Read`, `Glob`, `Grep`만. **Edit·Write 없음 → 수정 자체가 불가능**.
- **제약**:
  - 주관적 판단 금지 (문서에 명시된 규칙만 근거)
  - 기능 버그 검토 금지 (코드 작성자의 책임)
  - 설계 대안 제안 금지
- **반환 포맷**: `PASS` 또는 위반 YAML (file/rule/line_range/old_string/new_string/reason)

### 서브에이전트 정의 형식

각 `*.md` 파일은 frontmatter + 본문으로 구성:

```yaml
---
name: <에이전트명>
description: <언제 이 에이전트를 호출하는가 — 오케스트레이터의 라우팅 힌트>
model: sonnet           # 또는 opus, haiku
tools:                  # 생략 시 전체 도구 사용
  - Read
  - Glob
  - Grep
---

<에이전트의 역할·제약·출력 포맷 지시>
```

---

## 3. `skills/` — 스킬

특정 워크플로를 캡슐화한 재사용 가능한 스킬. 사용자의 요청 맥락에 따라 Claude가 자동으로 트리거한다.

### 현재 정의된 스킬

#### `implement/SKILL.md`

요구사항을 받아 `design-writer`(D) → `code-writer`(A) → `architecture-reviewer`(B) 루프를 돌리는 오케스트레이션 스킬.
"구현해줘", "추가해줘", "리팩토링" 등 코드 작업 요청 시 자동 트리거.
`references/` 디렉토리에 각 에이전트의 입출력 계약 문서가 있으며, 오케스트레이터가 에이전트 프롬프트를 구성할 때 사용한다.
세부 흐름은 [WORKFLOW.md](WORKFLOW.md) 참조.

### 스킬 정의 형식

```yaml
---
name: <스킬명>
description: <트리거 조건 및 기능 설명 — 구체적이고 "pushy"하게 작성>
---

<스킬 실행 로직>
```

- `description`이 트리거 조건 역할을 한다. 언제 이 스킬을 써야 하는지 명확히 기술.
- 본문은 메인 Claude에게 내리는 **절차적 지시**. Phase별로 분할 작성.

---

## 4. `settings.json` / `settings.local.json` — 운용 설정

### `settings.json` (팀 공유, 커밋 대상)

현재 정의:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/post-tool-use.sh" }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command", "command": "bash .claude/hooks/architectural-review-reminder.sh" }] }
    ]
  }
}
```

- **hooks** — 특정 이벤트에 셸 커맨드 실행. 위 예시는:
  - 파일 수정 후 자동 검사 훅
  - 세션 종료 직전 아키텍처 검토 리마인더

> 현재 `.claude/hooks/` 디렉토리의 스크립트는 별도로 추가 필요. 빈 훅 참조만 둔 상태라면 해당 동작은 no-op.

### `settings.local.json` (로컬 전용, 커밋 제외)

- 개인별 권한 허용/거부 리스트 (`allow`, `deny`)
- API 키 등 로컬 환경 고유 설정

---

## 커스터마이징

### 새 서브에이전트를 추가

1. `.claude/agents/<name>.md` 생성 — 역할·제약·반환 포맷 명시
2. 필요하면 `tools`로 도구 권한 제한
3. 오케스트레이션에 통합하려면 기존 스킬 수정 또는 새 스킬 추가

### 새 스킬을 추가

1. `.claude/skills/<name>/SKILL.md` 생성
2. frontmatter에 `name`, `description` 정의 — description이 트리거 조건
3. Phase별 절차를 본문에 작성 — Claude가 이 문서를 단계별로 따라간다

### 훅(Hook) 추가

1. `.claude/hooks/<name>.sh` 스크립트 작성
2. `settings.json`의 `hooks` 배열에 matcher와 command 등록
3. matcher 예: `Write|Edit` (도구 이름 정규식), 이벤트 타입: `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStop` 등

### 권한 허용 리스트 관리

프로젝트에서 자주 쓰는 명령을 미리 허용해 프롬프트를 줄이려면 `settings.local.json`에 등록.
에이전트가 허용되지 않은 명령을 시도하면 사용자에게 승인 요청이 뜬다.

---

## 핵심 원칙 (수정 시 지킬 것)

1. **서브에이전트의 tools는 최소 권한**으로. 특히 검토자는 Read 전용.
2. **커맨드의 allowed-tools는 화이트리스트**. 필요 없는 도구는 포함하지 않는다.
3. **CLAUDE.md는 링크 허브**. 규칙 본문은 `docs/` 하위에만.
4. **에이전트 정의에 문서 경로를 하드코딩**. 에이전트가 "어떤 문서를 읽어야 하는가"를 스스로 탐색하지 않게 한다.

---

## 참고

- [WORKFLOW.md](WORKFLOW.md) — 에이전트들이 실제 어떻게 상호작용하는지
- [PHILOSOPHY.md](PHILOSOPHY.md) — 왜 이렇게 쪼개고 제한하는가
- [DOCS_GUIDE.md](DOCS_GUIDE.md) — 에이전트가 참조하는 문서들
- [Claude Code 공식 문서 — Subagents](https://docs.claude.com/en/docs/claude-code/sub-agents)
- [Claude Code 공식 문서 — Skills](https://docs.claude.com/en/docs/claude-code/skills)
- [Claude Code 공식 문서 — Hooks](https://docs.claude.com/en/docs/claude-code/hooks)
