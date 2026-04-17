# Frontend Architecture

Backoffice 프론트엔드는 **Feature-Sliced Design(FSD)** 아키텍처를 기반으로 구성된다.

---

## 개요

FSD는 기능 단위로 코드를 슬라이스(slice)하여 관심사를 분리하고, 레이어 간 단방향 의존성을 강제하는 프론트엔드 아키텍처 방법론이다. 코드의 결합도를 낮추고 유지보수성을 높이는 것을 목표로 한다.

---

## 레이어 구성

총 6개의 레이어로 구성되며, 위에서 아래로 갈수록 하위 레이어이다.

```
app
pages
widgets
features
entities
shared
```

### app

프로젝트의 진입점이자 전역 설정을 담당하는 최상위 레이어.

- 라우팅 정의
- 글로벌 스타일 및 테마
- 최상위 프로바이더 (인증, 쿼리 클라이언트 등)
- 프로젝트 루트 구성 및 초기화

### pages

특정 URL에 매핑되는 개별 페이지를 정의하는 레이어.

- 라우팅 단위로 구성되는 최상위 UI 컴포넌트
- 복잡한 비즈니스 로직보다 UI 조합 및 인터페이스 로직 처리
- widgets, features, entities를 조합해 화면을 구성

### widgets

페이지 내에서 독립적으로 작동하는 큰 기능 단위의 레이어.

- 여러 페이지에서 재사용 가능한 대형 UI 블록
- features와 entities를 조합하여 구성
- 독립적으로 기능하는 컴포넌트 (헤더, 사이드바, 데이터 테이블 등)

### features

재사용 가능한 비즈니스 기능을 추상화하는 레이어.

- 사용자가 수행하는 단일 행위 단위 (로그인, 상품 등록 등)
- 독립적으로 동작 가능한 기능 블록
- 다양한 pages, widgets에서 재사용

### entities

프로젝트의 비즈니스 핵심 데이터를 다루는 레이어.

- 도메인 데이터 모델 및 타입 정의
- 데이터 조회 훅, API 연동 로직
- 다양한 레이어에서 공통으로 사용되는 도메인 로직

### shared

프로젝트 전반에서 범용적으로 재사용되는 리소스 레이어.

- 특정 비즈니스 또는 기능에 종속되지 않는 코드
- 공통 UI 컴포넌트, 유틸리티 함수, 상수, 타입
- HTTP 클라이언트, 환경변수, 공통 훅

---

## 단방향 의존성 규칙

각 레이어는 **자신보다 하위 레이어만 참조**할 수 있다. 동일 레이어 간 참조는 금지된다.

```
app       →  pages, widgets, features, entities, shared
pages     →  widgets, features, entities, shared
widgets   →  features, entities, shared
features  →  entities, shared
entities  →  shared
shared    →  (참조 없음)
```

### 규칙 도입 이유

- **결합도 감소**: 상위 레이어 변경이 하위 레이어로 전파되지 않는다.
- **변경 영향 범위 제한**: 특정 레이어의 변경이 다른 레이어로 확산되지 않는다.
- **코드 구조 명확화**: 의존 방향이 일관되어 코드 탐색 및 이해가 쉬워진다.
- **테스트 용이성**: 하위 레이어는 상위 레이어에 의존하지 않으므로 독립적으로 테스트 가능하다.

### 위반 사례

```ts
// 금지: features에서 pages를 import
import { SomePage } from '@/pages/some-page'

// 금지: entities에서 features를 import
import { useLogin } from '@/features/auth'

// 금지: shared에서 entities를 import
import { Product } from '@/entities/product'
```

---

## 슬라이스와 세그먼트

레이어 내부는 **슬라이스(도메인 단위)**로 나뉘고, 슬라이스는 **세그먼트(역할 단위)**로 구성된다.

```
features/
  auth/              ← 슬라이스 (도메인)
    ui/              ← 세그먼트 (UI 컴포넌트)
    model/           ← 세그먼트 (상태, 훅)
    api/             ← 세그먼트 (API 통신)
    index.ts         ← Public API (외부 노출 진입점)
```

각 슬라이스는 반드시 `index.ts`를 통해 외부에 노출할 인터페이스를 명시적으로 정의한다. 슬라이스 내부 구현은 외부에서 직접 접근하지 않는다.

```ts
// 허용: Public API를 통한 접근
import { LoginForm } from '@/features/auth'

// 금지: 내부 구현 직접 접근
import { LoginForm } from '@/features/auth/ui/LoginForm'
```
