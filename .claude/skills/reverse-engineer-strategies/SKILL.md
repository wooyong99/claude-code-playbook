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

---

## Step 1-D. 기존 문서 상태 파악 (레이어별)

출력 경로를 확인하여 **레이어별로** 기존 `strategies/` 파일 유무를 확인한다.
레이어마다 독립적으로 아래 두 가지 모드 중 하나를 결정한다.

### 레이어별 모드 결정

각 레이어에 대해:
- `{출력경로}/{레이어}/strategies/` 하위에 파일이 **하나라도 존재**하면 → **병합 대상 레이어**
- 파일이 **전혀 없으면** → **신규 생성 레이어** (사용자에게 묻지 않음)

레이어별 모드 맵을 기록한다:

```text
레이어별 처리 모드
- app        → 신규 생성 (기존 파일 없음)
- application → 병합 대상 (기존 파일 N개 발견)
- domain     → 신규 생성 (기존 파일 없음)
- storage    → 병합 대상 (기존 파일 N개 발견)
- external   → 신규 생성 (기존 파일 없음)
```

**병합 대상 레이어가 하나라도 있을 때만** 사용자에게 전체 병합 방식을 묻는다.

> "일부 레이어에 기존 strategies/ 문서가 있습니다.
> - [{레이어명}: N개 파일]
> 기존 파일이 있는 레이어를 어떻게 처리할까요?
> - [1] **병합** — 기존 내용과 코드 분석 결과를 대조하여, 일치하면 보존·부족한 항목만 추가하고, 코드와 불일치하는 내용은 수정합니다.
> - [2] **덮어쓰기** — 기존 파일을 삭제하고 코드 분석 결과로 새로 작성합니다."

기본값: **[1] 병합**

신규 생성 레이어는 선택과 무관하게 항상 새로 생성한다.

---

## Step 2. 프로젝트 구조 탐색

입력 받은 코드베이스 경로에서 아래 순서로 탐색한다.

### 2-1. 멀티 모듈 vs 단일 모듈 식별

먼저 대상 코드베이스가 **멀티 모듈인지 단일 모듈인지 식별**한다.

- `settings.gradle.kts`가 존재하고 `include(...)` 선언이 2개 이상이면 → **멀티 모듈**
- 그렇지 않으면 → **단일 모듈**

### 2-2. 구조 파악

**멀티 모듈인 경우**

`settings.gradle.kts`, 각 모듈의 `build.gradle.kts`, `src/main` 구조를 읽어 **레이어별 모듈 후보**를 식별한다.

```bash
cat {codebase_path}/settings.gradle.kts
find {codebase_path} -name "build.gradle.kts" | grep -v "build/" | head -20
find {codebase_path} -name "*.kt" -path "*/src/main/*" | head -50
```

**단일 모듈인 경우**

`src/main/kotlin` 이하 패키지와 디렉토리 구조를 읽어 **레이어별 디렉토리 후보**를 식별한다.

```bash
find {codebase_path}/src/main/kotlin -type d | head -40
find {codebase_path}/src/main/kotlin -name "*.kt" | head -50
```

### 2-3. 로컬 레이어 매핑

식별한 모듈·디렉토리를 플레이북의 개념 레이어에 매핑한다.

- `domain`, `application`, `storage`, `external`, `app` 같은 플레이북 개념 레이어를 먼저 떠올리되, 실제 프로젝트가 쓰는 **로컬 레이어 이름과 경계**를 우선 기록한다.
- 플레이북에 적힌 예시 레이어명과 정확히 일치하지 않더라도, **실제 코드에서 드러나는 역할과 책임**을 기준으로 매핑한다.
- 예를 들어 `app` 대신 `api`, `presentation`, `bootstrap`을 쓸 수 있고, `storage`와 `external`을 `infra` 또는 `infrastructure` 하위에 둘 수도 있다.
- 추측하지 말고 실제 클래스 역할, 어노테이션, 네이밍 패턴, 의존 방향을 근거로 판단한다.

매핑 결과는 아래 형식으로 기록한다.

```text
로컬 레이어 맵
- {로컬 이름} -> {개념 레이어}
- {로컬 이름} -> {개념 레이어}
- ...
```

---

## Step 3. 레이어별 코드 분석

각 레이어에 대해 [`references/layer-analysis-guide.md`](references/layer-analysis-guide.md)를 참고하여 분석한다.
레이어별 분석 대상, 확인할 패턴, 결과 기록 형식이 모두 해당 문서에 정의되어 있다.

각 레이어 분석 시 반드시 실제 파일을 읽어서 사실 기반으로 답한다. 추측 금지.

### 레거시/불명확 패턴 감지

분석 중 아래 신호를 발견하면 **레거시 코드베이스**로 간주하고 별도 메모를 남긴다:

- 서비스 클래스 명명이 혼재 (`Service`, `Manager`, `Handler`, `Helper` 동시 사용)
- `@Transactional`이 레이어 경계를 벗어난 클래스에 붙어 있음
- Controller에서 URL 동사 사용 (`/do-confirm`, `/create-order` 등)
- 도메인 로직이 Service 클래스에 직접 구현됨 (도메인 모델 없음)
- JPA Entity와 도메인 모델 미분리
- 순환 의존 또는 레이어 역방향 의존

레거시 코드베이스로 판단되면 **현재 패턴을 있는 그대로 문서화**하면서, Step 5 문서 생성 시 각 컨벤션 문서 하단에 `## 개선 제안` 섹션을 추가한다 (Step 5 참조).

---

## Step 4. 불확실한 부분 확인

분석 후 확실하지 않은 부분이 있다면 사용자에게 질문한다. 단, 코드에서 명확히 드러나는 사항은 질문하지 않는다.

예시 질문:
- "Auth 모듈이 두 가지 방식(JWT + Session)을 혼용하는 것 같은데, 주된 방식이 무엇인가요?"
- "QueryDsl 모듈은 있는데 실제로 사용하는 Repository가 보이지 않습니다. 다른 패턴을 쓰시나요?"

---

## Step 5. 문서 생성

분석 결과를 바탕으로 [`references/strategies-doc-templates.md`](references/strategies-doc-templates.md)를 참고하여 문서를 생성한다.

**레이어별 처리 모드 맵**(Step 1-D에서 작성)에 따라 레이어마다 아래 분기로 처리한다.

---

### [신규 생성 레이어]

기존 파일이 없는 레이어는 코드 분석 결과만을 바탕으로 처음부터 새로 작성한다.

---

### [병합 대상 레이어 + 병합 선택 시]

1. 기존 파일 목록을 확인한다
2. **각 파일을 읽고** 내용을 파악한다
3. 코드 분석 결과와 **항목별로 대조**한다:
   - **코드와 일치하는 내용** → 그대로 보존
   - **코드와 불일치하는 내용** (예: docs는 QueryDSL이라 했는데 코드는 JOOQ) → 코드 기반으로 수정
   - **플레이스홀더 `{...}` 형태** → 코드 분석으로 발견한 실제 값으로 채움
   - **코드에서 새로 발견된 패턴** → 해당 문서 또는 새 문서에 추가
4. 불일치 수정이 발생한 경우, 완료 보고(Step 6)의 "수정됨" 항목에 기록한다

> 왜 불일치를 수정하는가: strategies 문서의 목적은 "이 코드베이스가 실제로 선택한 전략"을 기술하는 것이다. 문서가 코드와 다른 내용을 담으면 팀원들이 잘못된 방향으로 개발하게 된다. 기존 문서가 사람이 직접 작성한 것이더라도, 코드가 그 내용을 따르지 않고 있다면 코드가 정답이다.

---

### [병합 대상 레이어 + 덮어쓰기 선택 시]

해당 레이어의 `strategies/` 디렉토리 내 파일을 모두 삭제하고, 코드 분석 결과만을 바탕으로 처음부터 새로 작성한다.

---

### 레거시 코드베이스 문서화

Step 3에서 레거시로 판단된 코드베이스의 경우, 각 컨벤션 문서 맨 하단에 `## 개선 제안` 섹션을 추가한다.

이 섹션의 목적은 "앞으로 이 코드를 어떻게 개선해나갈 수 있는지" 방향을 제시하는 것이다. 현재 패턴을 비판하는 것이 아니라, 팀이 점진적으로 나아갈 수 있는 실용적 개선 방향을 제안한다.

```markdown
## 개선 제안

> 현재 코드베이스에서 관찰된 패턴과 점진적 개선 방향.

### 관찰된 안티패턴

| 패턴 | 현재 코드 | 권장 방향 |
|-----|---------|---------|
| {안티패턴명} | {현재 코드 예시} | {개선 방향} |

### 점진적 개선 로드맵

1. **즉시 개선 가능**: {작은 변경으로 큰 효과를 낼 수 있는 항목}
2. **중기 개선**: {설계 변경이 필요하지만 단계적으로 가능한 항목}
3. **장기 개선**: {아키텍처 재설계가 필요한 항목}
```

---

## Step 5-B. 미사용 문서 제거 (병합 모드 전용)

병합 모드에서, `strategies/` 디렉토리에 있는 기존 파일 중 **코드 분석 결과 해당 컴포넌트 패턴이 코드베이스에 존재하지 않는 파일**을 제거한다.

### 제거 판단 기준

**모든 strategies/ 파일**을 대상으로 아래 절차로 확인한다. (특정 파일명 목록에 의존하지 않는다.)

1. 파일명에서 컴포넌트 키워드를 추출한다 (예: `flow-convention.md` → `Flow`, `querydsl-convention.md` → `QueryDSL`)
2. 코드베이스에서 해당 컴포넌트 타입의 클래스 또는 라이브러리 사용이 존재하는지 확인한다
3. 존재하지 않으면 → 제거 대상

확인 방법 예시:

| 문서 파일명 키워드 | 코드베이스 존재 여부 확인 방법 |
|-----------------|---------------------------|
| `flow` | `class \w+Flow` 패턴의 `@Service` / `@Component` 클래스 |
| `validator` | `@Component class \w+Validator` 패턴 |
| `mapper` | `@Component class \w+Mapper` 또는 `@Service class \w+Mapper` 패턴 |
| `querydsl` | `JPAQueryFactory` 또는 `QueryDsl` import 사용 여부 |
| `jooq` | `DSLContext` 또는 `jooq` import 사용 여부 |
| `resttemplate` | `RestTemplate` 사용 여부 |
| `okhttp` | `OkHttpClient` 사용 여부 |
| 기타 커스텀 파일 | 파일명에서 추론한 핵심 타입/라이브러리를 코드에서 grep으로 확인 |

### 처리 절차

제거 대상으로 판단된 파일마다 다음 순서로 처리한다.

1. **내용 검토**: 파일을 읽어 다른 문서에 아직 반영되지 않은 유용한 내용(금지 패턴, 체크리스트 항목 등)이 있는지 확인한다.
2. **내용 이전**: 반영 안 된 내용은 가장 관련성 높은 컨벤션 문서에 섹션으로 추가한다.
3. **파일 삭제**: 내용 이전이 완료된 파일을 삭제한다.
4. **링크 정리**: 삭제된 파일을 참조하는 `strategies/README.md`의 역할별 컴포넌트 표·Post-Work Verification 체크리스트에서 해당 링크를 제거하거나, 내용이 이전된 문서 링크로 교체한다.

### 제거하지 않는 경우

아래 중 하나에 해당하면 파일을 유지한다.
- 코드베이스에 해당 컴포넌트가 실제로 존재한다.
- 파일 내용이 "이 프로젝트에서 해당 컴포넌트를 쓰면 안 된다"는 명시적 금지 규칙 역할을 하며, 다른 문서에 이미 통합된 내용이 없다.

---

## Step 6. 완료 보고

생성/갱신된 파일 목록, 삭제된 파일 목록, 수정된 내용, 핵심 전략 결정 사항을 사용자에게 보고한다.

```
## 완료

### 생성 / 갱신

| 파일 | 발견한 전략 |
|-----|-----------|
| storage/strategies/README.md | JPA + QueryDsl |
| storage/strategies/storage-adapter-convention.md | {Entity}Adapter → JpaRepository + QueryDslRepository |
| app/strategies/README.md | JWT Stateless, Header-based multi-tenant |
| ...  | ... |

### 신규 추가 (디폴트 목록에 없던 패턴)

| 파일 | 발견한 패턴 | 근거 |
|-----|-----------|-----|
| application/strategies/scheduler-convention.md | @Scheduled 작업 공통 구조 | XxxScheduler 클래스 3개 발견 |
| ...  | ... | ... |

### 삭제 (미사용 문서)

| 삭제 파일 | 이유 | 내용 이전 위치 |
|---------|-----|-------------|
| application/strategies/flow-convention.md | Flow 클래스 미사용 | use-case-convention.md |
| ...  | ... | ... |

### 수정 (코드와 불일치하여 교체)

| 수정 전 내용 | 수정 후 내용 | 근거 |
|-----------|-----------|-----|
| storage: QueryDSL 기반 문서 | storage: JOOQ 기반 문서 | DSLContext 사용 확인 |
| ...  | ... | ... |

**추가 확인 필요**:
- {불확실해서 임시로 작성한 항목과 이유}
```
