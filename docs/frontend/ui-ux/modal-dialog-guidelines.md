# Modal & Dialog Guidelines

삭제 confirm, 경고 모달, 상세 모달, 바텀 시트 등 모달 관련 정책을 정의한다.

---

## 모달 유형

| 유형 | 목적 | 예시 |
|------|------|------|
| Confirm Dialog | 사용자 의도 재확인 | 삭제 확인, 비활성화 확인 |
| Alert Dialog | 중요 정보 전달 | 경고, 오류 안내 |
| Form Modal | 데이터 입력/수정 | 카테고리 수정, 간단한 등록 |
| Detail Modal | 데이터 상세 조회 | 주문 상세, 미리보기 |
| Bottom Sheet | 모바일 환경 액션 선택 | 정렬 옵션, 필터 선택 |

---

## Confirm Dialog 사용 기준

Confirm Dialog는 아래 조건 중 하나를 충족할 때 사용한다.

1. **되돌릴 수 없는 작업** — 영구 삭제, 초기화
2. **영향 범위가 큰 작업** — 다수의 데이터에 영향을 주는 일괄 처리
3. **의도하지 않은 클릭 가능성이 높은 위치**에 있는 destructive 버튼

단순 저장/수정처럼 되돌릴 수 있는 작업에는 confirm dialog를 사용하지 않는다.

### Confirm Dialog 구성 요소

```
[제목]
삭제하시겠습니까?

[본문]
'여름 신상품' 카테고리를 삭제합니다.
삭제된 데이터는 복구할 수 없습니다.

[버튼]
                       [취소]  [삭제]
```

- 제목: 수행하려는 동작을 명시 (예: "카테고리 삭제")
- 본문: 대상과 영향을 명시, 되돌릴 수 없음을 안내
- 취소 버튼: 항상 존재, secondary 스타일
- 확인 버튼: 동작을 명시하는 동사 사용 ("삭제", "비활성화" — "확인" 사용 금지)

---

## Destructive Action 문구 규칙

destructive 액션의 버튼과 문구는 아래 기준을 따른다.

| 항목 | 규칙 |
|------|------|
| 버튼 레이블 | 동사형으로 명시 ("삭제", "초기화", "비활성화") |
| 버튼 색상 | red 계열 (`bg-red-600`, `text-red-600`) |
| 버튼 위치 | 취소 버튼 우측 마지막에 배치 |
| 본문 문구 | "이 작업은 되돌릴 수 없습니다." 포함 |
| 영향 대상 | 대상 이름 또는 수량 명시 |

```
금지: [확인]
허용: [삭제]

금지: "계속하시겠습니까?"
허용: "삭제된 데이터는 복구할 수 없습니다."
```

---

## 모달 닫힘 조건

| 상황 | 닫힘 허용 여부 |
|------|---------------|
| 취소 버튼 클릭 | 항상 허용 |
| ESC 키 | 기본 허용 (단, submit 중 제한) |
| Overlay(배경) 클릭 | 기본 허용 (단, 미저장 폼 데이터 있을 시 확인 요구) |
| submit 진행 중 | 닫힘 차단 |
| 성공 후 | 자동 닫힘 |

### ESC / Overlay 클릭 허용 여부

- **Confirm Dialog**: ESC 및 overlay 클릭으로 닫힘 허용 (취소와 동일한 효과)
- **Form Modal**: 입력값이 없으면 ESC / overlay 클릭 허용. 입력값이 있으면 "저장하지 않고 닫으시겠습니까?" 확인 후 닫힘
- **Detail Modal**: ESC 및 overlay 클릭으로 닫힘 허용
- **Alert Dialog**: ESC로 닫힘 허용, overlay 클릭은 비허용 (중요 안내 전달 목적)

---

## Submit 중 닫힘 방지

API 요청 중(submit 진행 중)에는 모달 닫힘을 차단한다.

- ESC 키 이벤트 무시
- Overlay 클릭 무시
- 취소 버튼 disabled 처리
- 확인 버튼 로딩 상태로 전환

```
submit 완료 전:
[취소 — disabled]  [● 저장 중... — disabled]

submit 완료 후:
자동 닫힘 + 성공 토스트
```

---

## 모달 안에서 모달 중첩 제한

모달 위에 또 다른 모달을 여는 것은 원칙적으로 금지한다.

**허용 예외 — Confirm Dialog만 예외적으로 허용**

Form Modal에서 삭제 버튼을 누를 경우, 그 위에 Confirm Dialog를 여는 것은 허용한다.

```
Form Modal (수정)
  └── Confirm Dialog (삭제 확인)  ← 허용
```

**금지 패턴**

```
Form Modal
  └── 또 다른 Form Modal  ← 금지
  └── Detail Modal        ← 금지
```

모달 중첩이 필요한 경우 아래 대안을 검토한다.

| 상황 | 대안 |
|------|------|
| 모달 안에서 추가 정보 입력 | 모달 내 인라인 섹션으로 확장 |
| 모달 안에서 상세 조회 | 새 페이지 또는 슬라이드 오버 패널로 이동 |
| 복잡한 다단계 입력 | 모달 대신 페이지로 이동 |

---

## 접근성 기준

모달은 아래 접근성 요건을 충족해야 한다.

### 포커스 관리

- 모달이 열리면 포커스가 모달 내 첫 번째 포커스 가능 요소로 이동
- 모달이 닫히면 포커스가 모달을 열었던 원래 버튼으로 복귀
- 모달이 열려 있는 동안 포커스는 모달 내부에서만 순환 (Focus Trap)

### 키보드 접근성

- `Tab` / `Shift+Tab`: 모달 내 포커스 가능 요소 순환
- `ESC`: 모달 닫힘 (닫힘이 허용된 경우)
- `Enter`: 기본 액션 버튼 실행

### ARIA 속성

```html
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  aria-describedby="modal-description"
>
  <h2 id="modal-title">카테고리 삭제</h2>
  <p id="modal-description">삭제된 데이터는 복구할 수 없습니다.</p>
</div>
```

- `role="dialog"` 또는 `role="alertdialog"` (destructive / alert 용도)
- `aria-modal="true"` 로 스크린 리더의 포커스 범위를 모달로 제한
- `aria-labelledby`로 모달 제목 연결
- 배경(overlay)은 `aria-hidden="true"` 처리
