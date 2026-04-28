---
name: reverse-engineer-strategies
description: 기존 Kotlin + Spring Boot 백엔드 코드베이스를 분석하여 claude-code-playbook의 strategies/ 문서를 자동 생성하는 스킬. 레거시 프로젝트나 기존 코드베이스에 플레이북을 적용할 때, strategies/ 디렉토리 하위에 프로젝트 실제 구현 패턴을 반영한 컨벤션 문서를 만들어야 할 때 사용한다. "기존 코드 분석해서 strategies 문서 만들어줘", "레거시 코드베이스에서 전략 문서 추출", "코드 보고 strategies/ 채워줘", "기존 프로젝트 플레이북 적용해줘" 같은 요청에 반드시 이 스킬을 사용한다.
model: opus
---

# 기존 코드베이스 → Strategies 문서 역공학

기존 Kotlin + Spring Boot 백엔드 코드베이스를 읽어 각 아키텍처 레이어의 실제 구현 전략을 파악하고,
`docs/backend/architecture/*/strategies/` 하위 문서들을 자동으로 생성한다.

---

## 참고 문서

작업 전 아래 두 파일을 읽어 분석 방법과 출력 형식을 숙지한다.

- [`references/layer-analysis-guide.md`](references/layer-analysis-guide.md) — 레이어별 분석 방법 (어떤 파일을 읽고, 어떤 패턴을 찾아야 하는지)
- [`references/strategies-doc-templates.md`](references/strategies-doc-templates.md) — 출력 문서 형식 (README.md와 컨벤션 문서 템플릿)

---

## Step 1. 입력 파악

사용자에게 다음 두 가지를 확인한다.

**1-A. 분석 대상 코드베이스 경로**
> "분석할 코드베이스 경로를 알려주세요. (예: `/Users/me/projects/my-backend`)"

사용자가 현재 디렉토리라고 하거나 생략하면 CWD를 사용한다.

**1-B. 문서 출력 경로**
> "생성된 strategies/ 문서를 어디에 저장할까요?
> - [1] 현재 플레이북의 `docs/backend/architecture/`
> - [2] 대상 코드베이스 내 `docs/backend/architecture/`
> - [3] 직접 입력"

기본값: 현재 플레이북의 `docs/backend/architecture/`.

**1-C. 분석 대상 레이어 (선택 사항)**
> "특정 레이어만 분석할까요? (app / application / domain / storage / external)
> 지정하지 않으면 존재하는 레이어를 모두 분석합니다."

**1-D. 기존 문서 처리 방식 (출력 경로에 파일이 이미 있을 때)**

출력 경로를 확인하여 `strategies/` 하위에 파일이 하나라도 존재하면 사용자에게 묻는다.
파일이 없으면 이 질문은 생략한다.

> "출력 경로에 이미 strategies/ 문서가 있습니다. 어떻게 처리할까요?
> - [1] **병합** — 기존 내용을 읽어 커스터마이징된 부분은 보존하고, 미완성·누락된 항목만 갱신합니다.
> - [2] **덮어쓰기** — 기존 파일을 삭제하고 코드 분석 결과로 새로 작성합니다."

기본값: **[1] 병합**

---

## Step 2. 프로젝트 구조 탐색

입력 받은 코드베이스 경로에서 아래 순서로 탐색한다.

### 2-1. 최상위 구조 파악

```bash
ls {codebase_path}
find {codebase_path} -name "build.gradle.kts" -o -name "settings.gradle.kts" | head -20
find {codebase_path} -name "*.kt" -path "*/src/main/*" | head -50
```

멀티모듈 구조라면 `settings.gradle.kts`에서 모듈 목록을 읽어 각 모듈의 역할을 파악한다.

### 2-2. 레이어별 모듈 매핑

아래 기준으로 모듈을 레이어에 대응시킨다.

| 레이어 | 모듈명 패턴 | 확인 방법 |
|-------|-----------|---------|
| app | `*-app`, `:app:*`, `web`, `api` | Controller, @SpringBootApplication 위치 |
| application | `*-application`, `:core:application`, `service` | @Service + UseCase/Flow 패턴 |
| domain | `*-domain`, `:core:domain`, `core` | 순수 Kotlin 클래스, 도메인 예외 |
| storage | `*-infra`, `:infra:storage`, `infrastructure` | @Entity, JpaRepository |
| external | `*-external`, `:infra:external`, `client` | @FeignClient, WebClient, RestTemplate |

레이어가 하나의 모듈에 혼재하는 경우 패키지 구조로 판단한다.

---

## Step 3. 레이어별 코드 분석

각 레이어에 대해 `references/layer-analysis-guide.md`의 해당 섹션을 참고하여 분석한다.

분석 순서:
1. **domain** (의존 없음, 순수 비즈니스 로직)
2. **application** (domain 의존)
3. **storage** (application Port 구현)
4. **external** (application Port 구현)
5. **app** (최외곽, HTTP/Security)

각 레이어 분석 시 반드시 실제 파일을 읽어서 사실 기반으로 답한다. 추측 금지.

### 분석 결과 기록 형식

레이어마다 분석이 끝나면 아래 형식으로 내부 메모를 정리한다.

```
[{LAYER}] 분석 결과
전략: {구체적인 전략명 — 예: JPA + QueryDsl, JWT Stateless}
이유: {코드에서 발견한 근거 — 예: QueryDslRepository 패턴, JwtTokenProvider 클래스}
컴포넌트:
  - {역할}: {클래스명 예시} → 컨벤션 문서: {파일명}
  - ...
불확실한 부분: {확인 필요한 항목}
```

---

## Step 4. 불확실한 부분 확인

분석 후 확실하지 않은 부분이 있다면 사용자에게 질문한다. 단, 코드에서 명확히 드러나는 사항은 질문하지 않는다.

예시 질문:
- "Auth 모듈이 두 가지 방식(JWT + Session)을 혼용하는 것 같은데, 주된 방식이 무엇인가요?"
- "QueryDsl 모듈은 있는데 실제로 사용하는 Repository가 보이지 않습니다. 다른 패턴을 쓰시나요?"

---

## Step 5. 문서 생성

분석 결과를 바탕으로 `references/strategies-doc-templates.md`의 템플릿에 따라 문서를 생성한다.

### 생성 대상

각 레이어마다:
- **`strategies/README.md`** — 이 프로젝트의 전략 요약 (필수)
- **`strategies/{component}-convention.md`** — 발견된 컴포넌트별 컨벤션 (발견된 것만)

### 파일 경로

```
{출력경로}/
├── app/strategies/
│   ├── README.md
│   ├── api-convention.md            (Controller/DTO 패턴 발견 시)
│   ├── rest-design-convention.md    (REST 설계 규칙 발견 시)
│   ├── exception-handling-convention.md  (GlobalExceptionHandler 발견 시)
│   └── file-structure.md            (패키지 구조가 명확할 때)
├── application/strategies/
│   ├── README.md
│   ├── use-case-convention.md       (UseCase 패턴 발견 시)
│   ├── flow-convention.md           (Flow/Orchestrator 발견 시)
│   ├── validator-convention.md      (Validator 발견 시)
│   ├── handler-convention.md        (Handler/ACL 발견 시)
│   ├── policy-convention.md         (Policy/Strategy 발견 시)
│   ├── event-handler-convention.md  (EventHandler 발견 시)
│   └── mapper-convention.md         (Mapper/Assembler 발견 시)
├── domain/strategies/
│   ├── README.md
│   ├── domain-model-convention.md   (Entity/VO 패턴 발견 시)
│   └── exception-convention.md      (도메인 예외 패턴 발견 시)
├── storage/strategies/
│   ├── README.md
│   ├── storage-adapter-convention.md  (Adapter 패턴 발견 시)
│   └── {orm}-convention.md            (QueryDsl/JOOQ 등 발견 시)
└── external/strategies/
    ├── README.md
    ├── api-client-{http-client}.md    (HTTP 클라이언트 발견 시)
    └── api-client-logging.md          (로깅 패턴 발견 시)
```

### 기존 파일 처리

Step 1-D에서 선택한 방식에 따라 분기한다.

**[1] 병합 선택 시**
1. 기존 파일을 먼저 읽는다
2. 커스터마이징된 내용(플레이스홀더가 아닌 실제 기술 내용)은 그대로 보존한다
3. `{플레이스홀더}` 형태로 남은 미완성 항목과 코드 분석으로 새로 발견된 항목만 추가·갱신한다

**[2] 덮어쓰기 선택 시**
1. 해당 `strategies/` 디렉토리 내 파일을 모두 삭제한다
2. 코드 분석 결과만을 바탕으로 처음부터 새로 작성한다

---

## Step 5-B. 미사용 문서 제거 (병합 모드 전용)

병합 모드에서, `strategies/` 디렉토리에 있는 기존 파일 중 **코드 분석 결과 해당 컴포넌트 패턴이 코드베이스에 존재하지 않는 파일**을 제거한다.

### 제거 판단 기준

문서 파일명이 나타내는 컴포넌트 타입의 실제 클래스(Spring Bean)가 코드베이스에 없는 경우 제거 대상이다.

| 문서 | 존재 여부 확인 방법 |
|-----|-----------------|
| `flow-convention.md` | `class \w+Flow` 패턴의 `@Service` / `@Component` 클래스 존재 여부 |
| `validator-convention.md` | `@Component class \w+Validator` 패턴 존재 여부 |
| `mapper-convention.md` | `@Component class \w+Mapper` / `@Service class \w+Mapper` 패턴 존재 여부 |

### 처리 절차

제거 대상으로 판단된 파일마다 다음 순서로 처리한다.

1. **내용 검토**: 파일을 읽어 다른 문서에 아직 반영되지 않은 유용한 내용(금지 패턴, 체크리스트 항목 등)이 있는지 확인한다.
2. **내용 이전**: 반영 안 된 내용은 가장 관련성 높은 컨벤션 문서에 섹션으로 추가한다 (예: `use-case-convention.md`의 "금지 패턴" 또는 "체크리스트").
3. **파일 삭제**: 내용 이전이 완료된 파일을 삭제한다.
4. **링크 정리**: 삭제된 파일을 참조하는 `strategies/README.md`의 역할별 컴포넌트 표·Post-Work Verification 체크리스트에서 해당 링크를 제거하거나, 내용이 이전된 문서 링크로 교체한다.

### 제거하지 않는 경우

아래 중 하나에 해당하면 파일을 유지한다.
- 코드베이스에 해당 컴포넌트가 실제로 존재한다.
- 파일 내용이 "이 프로젝트에서 해당 컴포넌트를 쓰면 안 된다"는 명시적 금지 규칙 역할을 하며, 다른 문서에 이미 통합된 내용이 없다.

---

## Step 6. 완료 보고

생성/갱신된 파일 목록, 삭제된 파일 목록, 핵심 전략 결정 사항을 사용자에게 보고한다.

```
## 완료

### 생성 / 갱신

| 파일 | 발견한 전략 |
|-----|-----------|
| storage/strategies/README.md | JPA + QueryDsl |
| storage/strategies/storage-adapter-convention.md | {Entity}Adapter → JpaRepository + QueryDslRepository |
| app/strategies/README.md | JWT Stateless, Header-based multi-tenant |
| ...  | ... |

### 삭제 (미사용 문서)

| 삭제 파일 | 이유 | 내용 이전 위치 |
|---------|-----|-------------|
| application/strategies/flow-convention.md | Flow 클래스 미사용 | use-case-convention.md |
| ...  | ... | ... |

**추가 확인 필요**:
- {불확실해서 임시로 작성한 항목과 이유}
```
