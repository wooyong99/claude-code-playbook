# Frontend

{프로젝트명} 프론트엔드 문서. 아키텍처, 컨벤션, 성능, UI/UX 가이드라인으로 구성된다.

---

## 디렉토리 구조

```
docs/frontend/
├── architecture/   # FSD 아키텍처, 폴더 구조, 상태 관리
├── conventions/    # 코드/네이밍/컴포넌트/API 컨벤션
├── performance/    # 렌더링/캐싱/리스트 최적화
└── ui-ux/          # UI 원칙, UX 가이드라인, 로딩·피드백, 모달
```

---

## 시작하기

- [Getting Started](getting-started.md) — 기술 스택, 프로젝트 구성, 로컬 실행

---

## 아키텍처 ([`architecture/`](architecture/))

Feature-Sliced Design(FSD) 기반. 레이어 간 단방향 의존성을 강제한다.

| 문서 | 설명 |
|------|------|
| [frontend-architecture](architecture/frontend-architecture.md) | FSD 개요, 레이어 정의, 단방향 의존성 규칙 |
| [folder-structure](architecture/folder-structure.md) | 패키지 구조, 세그먼트 역할, 네이밍 규칙 |
| [state-management](architecture/state-management.md) | 전역 상태 기준, props drilling 방지, derived state, 캐시 invalidation |

---

## 컨벤션 ([`conventions/`](conventions/))

| 문서 | 설명 |
|------|------|
| [code-conventions](conventions/code-conventions.md) | import 순서, type/interface 선택, enum 대신 const object, 함수 선언 방식 |
| [naming-conventions](conventions/naming-conventions.md) | 파일/컴포넌트/타입/변수/훅/상수/쿼리 키 네이밍 |
| [component-conventions](conventions/component-conventions.md) | 컴포넌트 책임 범위, 비즈니스 로직 분리, props 설계 |
| [api-conventions](conventions/api-conventions.md) | API 계층 구조, HTTP 클라이언트 설정, 함수/훅 네이밍, 에러 처리 |

---

## 성능 ([`performance/`](performance/))

| 문서 | 설명 |
|------|------|
| [rendering-guidelines](performance/rendering-guidelines.md) | re-render 방지, key 안정성, memo/useMemo/useCallback 기준 |
| [caching-strategy](performance/caching-strategy.md) | staleTime/gcTime, 쿼리 키 설계, prefetch, invalidation |
| [list-optimization](performance/list-optimization.md) | 페이지네이션, debounce, 가상 스크롤, 무한 스크롤 |

---

## UI / UX ([`ui-ux/`](ui-ux/))

| 문서 | 설명 |
|------|------|
| [ui-principles](ui-ux/ui-principles.md) | 스켈레톤 표시 기준, 빈 화면, 버튼 위치, 여백, 반응형, CTA 우선순위 |
| [ux-guidelines](ui-ux/ux-guidelines.md) | 삭제 confirm, 중복 클릭 방지, 성공/실패 피드백, destructive 구분 |
| [loading-and-feedback](ui-ux/loading-and-feedback.md) | 스켈레톤/버튼 로딩/블로킹 기준, 저장·삭제 피드백, toast 규칙 |
| [modal-dialog-guidelines](ui-ux/modal-dialog-guidelines.md) | confirm dialog, destructive 문구, 닫힘 조건, ESC/overlay, 접근성 |
