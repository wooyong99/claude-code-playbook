# Code Writer — Input / Output Contract

`implement` 스킬이 `code-writer` 서브에이전트와 주고받는 인터페이스 규격.  
에이전트 파일(`code-writer.md`)이 아닌 이 문서가 입출력 포맷의 단일 출처다.

---

## Input

### Type 1 — 신규 구현

오케스트레이터가 아래 형식으로 프롬프트를 구성해 전달한다.

```
[마일스톤]: {마일스톤 제목}
[요구사항]:
{구체적 범위·목표}

[프로젝트 컨텍스트]:
  - Kotlin + Spring Boot 멀티모듈, 멀티테넌트 e-커머스 플랫폼
  - 의존 방향: app → application → domain ← infra
  - 관련 레이어: {app/application/domain/storage 중 해당}
  - 관련 도메인: {도메인명}

[출력 규격]: 이 문서(.claude/skills/implement/references/code-writer-contract.md) — Output > Type 1 그대로.
```

### Type 2 — 위반 수정

```
[수정 작업]: 이전 작업에 대한 아키텍처 검토에서 아래 위반이 발견됐습니다.
각 위반마다 제시된 old_string → new_string을 **정확히 그대로** 적용하세요.

{B가 반환한 위반 YAML 원문}

[규칙]:
  - old_string을 임의로 해석·수정하지 말 것.
  - old_string이 파일에서 매칭되지 않으면 실패로 보고.
  - 모든 Edit 적용 후 컴파일 성공 확인.

[출력 규격]: 이 문서(.claude/skills/implement/references/code-writer-contract.md) — Output > Type 2 그대로.
```

---

## Output

작업 완료 후 반드시 아래 포맷 중 하나로 반환한다. 불필요한 서술 없이 간결하게.

### Type 1 (신규 구현)

```
## 완료 요약

### 변경 파일
- <절대 경로 1>: <1~2줄 설명>
- <절대 경로 2>: <1~2줄 설명>
...

### 핵심 설계 결정
- <결정 1>
- <결정 2>

### 빌드/테스트 결과
- 컴파일: 성공 / 실패 (실패 시 에러)
- 테스트: <통과 수>/<총 수> (실패가 있으면 파일명·테스트명)

### 불확실한 부분
- <있다면 기재. 없으면 "없음">
```

### Type 2 (위반 수정)

```
## 수정 적용 결과

### 적용 성공
- <file, rule>: 적용됨
- ...

### 적용 실패
- <file, rule>: 실패 이유 (old_string 매칭 안됨 / 기타 상세)
- ... (없으면 "없음")

### 빌드 결과
- 컴파일: 성공 / 실패 (실패 시 에러)
```
