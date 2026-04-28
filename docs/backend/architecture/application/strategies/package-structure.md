# Application 모듈 패키지 구조

> **[로컬 컨벤션]** 이 문서는 이 프로젝트의 [구현 전략](README.md)(UseCase + Flow)을 채택한 패키지 레이아웃 규칙이다.
> 다른 전략을 선택한 프로젝트는 자체 패키지 구조를 정의한다.

## 원칙

- 패키지 구조는 가시성 정책이다. 진입점(UseCase, EventHandler)은 도메인 루트에 노출하고, 구현 세부사항(Flow, Validator, Policy)은 서브패키지로 격리함으로써 외부에서 볼 것과 감출 것을 구분한다.
- 패키지는 실제 필요할 때 생성한다. Validator, Policy가 없으면 해당 서브패키지를 미리 만들지 않는다.
- 위치가 역할을 드러낸다. 어느 패키지에 있는지만 보고도 해당 컴포넌트의 계층과 역할을 파악할 수 있어야 한다.

---

`:core:application` 모듈의 디렉토리 레이아웃과 각 구성 요소의 위치 규칙을 정의한다.

---

## 규칙

**진입점(UseCase, EventHandler)은 도메인 패키지 루트에 flat 노출, 구현 세부사항(Flow, Validator, Policy)은 서브패키지로 격리한다.**

```
:core:application/
└── src/main/kotlin/com/wooyong/demo/core/application/
    ├── {domain}/
    │   ├── {Action}{Entity}UseCase.kt     ← UseCase: flat 노출 (진입점)
    │   ├── {Entity}EventHandler.kt        ← EventHandler: flat 노출
    │   ├── flow/
    │   │   └── {Entity}{Action}Flow.kt
    │   ├── validator/
    │   │   └── {Entity}Validator.kt
    │   ├── policy/
    │   │   └── {Entity}SomePolicy.kt
    │   ├── dto/
    │   │   └── {Action}{Entity}.kt
    │   ├── mapper/
    │   │   └── {Entity}DtoMapper.kt
    │   └── port/
    │       └── {Entity}Repository.kt
    │
    └── file/
        ├── AttachedFileHandler.kt
        └── port/
            └── FileStoragePort.kt
```

| 구성 요소 | 위치 | 이유 |
|-----------|------|------|
| UseCase | `{domain}/` flat | 진입점이므로 바로 노출 |
| EventHandler | `{domain}/` flat | UseCase와 동격 |
| Flow | `{domain}/flow/` | 실행 구현 세부사항 |
| Validator | `{domain}/validator/` | 검증 전용 컴포넌트 |
| Policy | `{domain}/policy/` | 인터페이스 + 순수 구현체 모음 |
| Handler | 해당 개념 도메인 패키지 flat | 여러 도메인에서 재사용 |

- Validator, Policy가 없으면 해당 서브패키지를 만들지 않는다.
- Flow가 1개뿐이어도 `flow/` 서브패키지로 분리한다.

---

## 체크리스트

- [ ] UseCase와 EventHandler가 도메인 패키지 루트에 flat 배치됐는가?
- [ ] Flow / Validator / Policy가 각각의 서브패키지에 위치하는가?
- [ ] Validator, Policy가 없는 경우 서브패키지를 생성하지 않았는가?
