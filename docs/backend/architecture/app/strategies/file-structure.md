# App 계층 — 파일 구조

---

## 핵심 규칙

**app 계층은 표현 계층 전역 관심사를 담는 `common/`과 도메인별 표현 객체를 담는 `{domain}/`으로 구성한다. 변환 Extension 파일(`{Domain}DtoExtension.kt`)은 반드시 `dto/` 밖 도메인 패키지 레벨에 둔다.**

파일 구조 문서는 **어디에 배치하는가**를 정의한다. 각 디렉토리가 어떤 책임을 가져야 하는지는 [common.md](common.md) 같은 개별 전략 문서에서 다룬다.

---

## 개요

| 디렉토리 | 역할 |
|---------|------|
| `common/` | 표현 계층 전역 관심사. 특정 도메인에 의존하지 않는다 |
| `{domain}/` | 도메인별 표현 객체와 진입점 |

---

## 전체 구조 트리

> `common/` 내부의 세부 서브패키지 구성은 전략에 따라 달라질 수 있다. 이 문서는 상위 배치 규칙만 정의하고, `common/` 내부 책임 기준은 [common.md](common.md)를 따른다.

```
{app-module}/
├── common/
│   └── ...              ← 공통 표현 계층 구성요소
└── {domain}/
    ├── {Entity}Controller.kt
    ├── {Domain}DtoExtension.kt   ← dto/ 밖에 위치
    └── dto/
        ├── {Domain}Requests.kt
        └── {Domain}Responses.kt  ← 필요 시만
```

---

## `{domain}/` 규칙

모든 도메인별 기능은 반드시 `{domain}/` 하위에 위치한다.

- `{domain}`은 단수형 소문자 (예: `auth`, `partner`, `catalog`, `inventory`, `stock`)
- Controller와 Extension은 `dto/` 밖 최상위에 평탄(flat) 배치한다.

### Controller

| 항목 | 규칙 |
|------|------|
| 위치 | `{domain}/{Entity}Controller.kt` |
| 역할 | 요청 수신 → UseCase 위임 → 응답 반환만 담당 |
| 금지 | 비즈니스 로직, 도메인 객체 직접 조작 |

### Extension

| 항목 | 규칙 |
|------|------|
| 위치 | `{domain}/{Domain}DtoExtension.kt` (`dto/` 바깥) |
| 역할 | Request → Command, Result → Response 변환 |
| 금지 | `dto/` 패키지 내부 배치 |

### DTO

| 항목 | 규칙 |
|------|------|
| Request 파일 | `dto/{Domain}Requests.kt` (복수). 1개면 단수형 |
| Response 파일 | `dto/{Domain}Responses.kt` (필요 시만) |
| 금지 | `toCommand()` 등 변환 로직을 DTO 내부에 포함 |

---

## 금지 사항

- `common/`과 `{domain}/`의 배치 경계를 무너뜨리지 않는다.
- 도메인 패키지 간 직접 참조를 허용하지 않는다.
- `{Domain}DtoExtension.kt`를 `dto/` 패키지 안에 배치하지 않는다.
- DTO 내부에 `toCommand()` 등 변환 로직을 포함하지 않는다.
- app 모듈 간 코드를 직접 공유하지 않는다 — 공유 모듈을 경유한다.

---

## 체크리스트

- [ ] `{Domain}DtoExtension.kt`가 `dto/` 밖 `{domain}/` 패키지 레벨에 위치하는가?
- [ ] Controller가 `{domain}/` 최상위에 flat 배치됐는가?
- [ ] 공통 구성요소와 도메인별 구성요소가 위치상 명확히 분리됐는가?
- [ ] 도메인 패키지 간 직접 참조가 없는가?
- [ ] DTO 내부에 변환 로직이 없는가?
