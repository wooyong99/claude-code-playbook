# Architecture Reviewer — Input / Output Contract

`implement` 스킬이 `architecture-reviewer` 서브에이전트와 주고받는 인터페이스 규격.  
에이전트 파일(`architecture-reviewer.md`)이 아닌 이 문서가 입출력 포맷의 단일 출처다.

---

## Input

오케스트레이터가 아래 형식으로 프롬프트를 구성해 전달한다.

```
[마일스톤]: {마일스톤 제목}
[검토 대상 파일]:
{A가 반환한 절대 경로 목록}

[추가 컨텍스트]: 이번 마일스톤에서 A가 집중한 설계 결정 요약
{A의 "핵심 설계 결정" 섹션 그대로}

[체크포인트 파일]: .claude/checkpoints/arch-reviewer-{run_id}-M{n}-r{iter}.md

[출력 규격]: 이 문서(.claude/skills/implement/references/architecture-reviewer-contract.md) — Output 섹션 그대로.
```

체크포인트 재호출 시 아래 필드가 추가된다:

```
[체크포인트]: [체크포인트 파일] 경로 참조.
완료된 파일은 건너뛰고 남은 파일부터 이어서 검토.
```

---

## Output

### Case A: 위반 없음

단일 토큰만 출력:

```
PASS
```

`PASS` 단독. 다른 문장·설명·인사말 금지.

### Case B: 위반 존재

각 위반을 아래 YAML 블록으로 반환. 여러 위반은 연속된 블록으로:

```yaml
- file: <절대 경로>
  rule: <문서명>:<규칙 또는 체크리스트 항목>
  line_range: <start-end>
  reason: <1줄 근거 + 참조 문서 경로>
```

### 필드 규칙

- `file`: 절대 경로 (상대 경로 금지)
- `rule`: 형식 `<문서명>:<항목>`
  - 예: `app-layer-guidelines.md:Controller 체크리스트 "@Valid가 Request DTO에 적용됐는가"`
  - 예: `logging.md:LogExtension 확장 함수 사용 규정`
- `line_range`: 시작-끝 라인 (예: `45-52`)
- `reason`: 1줄 근거 + 참조 문서 경로 (예: `reason: Request DTO에 toCommand() 로직 포함. app-layer-guidelines.md Coding Rules 2번.`)

### Case C: 컨텍스트 체크포인트

컨텍스트 윈도우가 65% 이상 소모된 경우, 출력 **첫 줄**에 아래 신호를 출력한 뒤 완료된 파일의 결과(PASS 또는 위반 YAML)를 이어서 작성한다:

```
CONTEXT_CHECKPOINT: {[체크포인트 파일] 경로}
```

이후 완료된 파일 수만큼의 정상 출력 포맷을 작성한다.

### 절대 출력하지 말 것

- 인사말·결론 문구 (예: "아래는 위반 목록입니다", "검토 완료")
- Markdown 헤더·본문
- 코드 블록 펜스(` ``` `) 외 YAML 데이터
- PASS, YAML, CONTEXT_CHECKPOINT 외의 어떤 텍스트

---

## 출력 예시

### 예시 1 — PASS

```
PASS
```

### 예시 2 — 위반 2건

```yaml
- file: /path/to/backend/app/backoffice/src/main/kotlin/com/example/backoffice/product/ProductController.kt
  rule: app-layer-guidelines.md:Controller 체크리스트 "@Valid가 Request DTO에 적용됐는가"
  line_range: 52-56
  reason: Request DTO에 @Valid 누락. docs/backend/architecture/app/app-layer-guidelines.md Post-Work Verification - Controller 섹션.

- file: /path/to/backend/core/application/src/main/kotlin/com/example/application/product/CreateProductUseCase.kt
  rule: logging.md:LogExtension 확장 함수 사용 규정
  line_range: 14-15
  reason: raw LoggerFactory 사용 + [SCOPE] 태그 누락. docs/backend/policies/logging.md Kotlin 사용 예시 및 안티 패턴.
```

---

## Self-check (출력 전 확인)

- [ ] 모든 지적이 **문서에 명시된 규칙**에 근거하는가? (추측·선호 배제)
- [ ] 출력이 PASS 또는 YAML 외의 텍스트를 포함하지 않는가?
- [ ] 기능 정확성·버그 관련 지적을 포함하지 않았는가?
- [ ] 설계 대안·선호 기반 제안을 포함하지 않았는가?
