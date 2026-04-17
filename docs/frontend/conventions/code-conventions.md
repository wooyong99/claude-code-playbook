# Code Conventions

코드 스타일과 작성 방식에 대한 규칙.

---

## Import 순서

import 구문은 아래 순서로 그룹을 나누고, 각 그룹 사이에 빈 줄 하나를 둔다.

```ts
// 1. React 관련 라이브러리
import { useState, useCallback } from 'react'
import { useNavigate } from 'react-router-dom'

// 2. 외부 라이브러리
import { useQuery } from '@tanstack/react-query'
import { z } from 'zod'

// 3. 내부 모듈 — 상위 레이어부터 하위 레이어 순
import { CategoryTable } from '@/widgets/category'
import { useCategoryDelete } from '@/features/category'
import { categoryKeys } from '@/entities/category'
import { Button, Modal } from '@/shared/ui'

// 4. 상대 경로 (같은 슬라이스 내부)
import { CategoryFormSchema } from './schema'
import type { CategoryFormValues } from './types'
```

### 규칙 요약

| 그룹 | 내용 |
|------|------|
| 1 | `react`, `react-dom`, `react-router-dom` 등 React 생태계 |
| 2 | `@tanstack/*`, `axios`, `zod` 등 서드파티 라이브러리 |
| 3 | `@/` path alias를 사용하는 내부 모듈 |
| 4 | `./`, `../` 상대 경로 모듈 |

- type import는 `import type` 구문으로 값 import와 분리한다.
- 같은 그룹 내 순서는 알파벳 순이 아닌 의존 계층 순(상위 → 하위)으로 정렬한다.

---

## 타입 정의

### type vs interface

| 대상 | 선택 | 이유 |
|------|------|------|
| 컴포넌트 Props | `interface` | 확장 가능성, 도구 지원 |
| API 응답/요청 타입 | `type` | 유니온, 인터섹션 조합에 유리 |
| 도메인 모델 | `type` | 불변 구조 표현에 적합 |
| 유틸리티 타입 조합 | `type` | Mapped/Conditional type 사용 필요 |
| 확장이 필요한 공통 구조 | `interface` | `extends`로 구조 상속 |

```ts
// Props는 interface
interface CategoryRowProps {
  category: Category
  onEdit: (id: string) => void
  onDelete: (id: string) => void
}

// 도메인 모델, API 타입은 type
type Category = {
  id: string
  name: string
  status: 'ACTIVE' | 'INACTIVE'
}

type CategoryListResponse = {
  items: Category[]
  totalCount: number
  page: number
}
```

### enum 대신 const object + as const 사용

TypeScript `enum`은 런타임 코드를 생성하고 tree-shaking이 되지 않는다. `const` 객체와 `as const`를 사용한다.

```ts
// 금지
enum CategoryStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
}

// 허용
const CATEGORY_STATUS = {
  ACTIVE: 'ACTIVE',
  INACTIVE: 'INACTIVE',
} as const

type CategoryStatus = typeof CATEGORY_STATUS[keyof typeof CATEGORY_STATUS]
// → 'ACTIVE' | 'INACTIVE'
```

### 유틸리티 타입 활용

반복적인 타입 정의 대신 TypeScript 내장 유틸리티 타입을 적극 활용한다.

```ts
// 수정 요청 타입은 생성 타입에서 파생
type CreateCategoryRequest = { name: string; description: string }
type UpdateCategoryRequest = Partial<CreateCategoryRequest> & { id: string }

// API 응답에서 특정 필드만 추출
type CategorySummary = Pick<Category, 'id' | 'name'>
```

---

## 함수 정의

### 컴포넌트가 아닌 함수: 함수 선언문 선호

일반 함수(유틸리티, 핸들러, 훅)는 함수 선언문으로 정의한다. 호이스팅이 적용되어 파일 내 배치 순서 의존성이 줄어든다.

```ts
// 권장: 함수 선언문
function formatPrice(price: number): string {
  return price.toLocaleString('ko-KR') + '원'
}

// 비권장: 화살표 함수 (콜백, 인라인 제외)
const formatPrice = (price: number): string => {
  return price.toLocaleString('ko-KR') + '원'
}
```

예외: 콜백으로 전달되는 함수, 클로저가 필요한 경우, 모듈 최상위에서 즉시 `export`하는 경우는 화살표 함수 허용.

### 컴포넌트: 화살표 함수 또는 함수 선언문 모두 허용

```tsx
// 허용: 화살표 함수
const CategoryRow = ({ category }: CategoryRowProps) => {
  return <tr>...</tr>
}

// 허용: 함수 선언문
function CategoryRow({ category }: CategoryRowProps) {
  return <tr>...</tr>
}
```

프로젝트 내에서 한 가지 방식으로 통일한다. 혼용하지 않는다.

### 커스텀 훅: 함수 선언문

```ts
// 권장
function useCategoryList(params: CategoryListParams) {
  return useQuery({ ... })
}

export { useCategoryList }
```

---

## 조건부 표현

### Optional Chaining / Nullish Coalescing 적극 활용

```ts
// 금지
const name = category && category.name ? category.name : '미설정'

// 허용
const name = category?.name ?? '미설정'
```

### 삼항 연산자 사용 기준

삼항 연산자는 단일 값 분기에만 사용한다. 중첩 삼항은 금지한다.

```tsx
// 허용: 단순 값 분기
const label = isActive ? '활성' : '비활성'

// 금지: 중첩 삼항
const label = isActive ? '활성' : isDeleted ? '삭제됨' : '비활성'

// 허용: 중첩이 필요한 경우 함수로 분리
function getStatusLabel(status: CategoryStatus): string {
  if (status === 'ACTIVE') return '활성'
  if (status === 'DELETED') return '삭제됨'
  return '비활성'
}
```

---

## 기타 규칙

### 매직 넘버/문자열 금지

의미 있는 숫자나 문자열은 상수로 선언한다.

```ts
// 금지
if (page > 20) { ... }

// 허용
const MAX_PAGE_SIZE = 20
if (page > MAX_PAGE_SIZE) { ... }
```

### 단언(as) 최소화

TypeScript 타입 단언(`as`)은 타입 안전성을 우회한다. 타입 가드로 대체한다.

```ts
// 금지
const category = data as Category

// 허용: 타입 가드 사용
function isCategory(value: unknown): value is Category {
  return typeof value === 'object' && value !== null && 'id' in value
}
```

불가피하게 `as`를 써야 할 경우 주석으로 이유를 명시한다.

### Non-null 단언(!) 금지

`!` 단언 대신 옵셔널 체이닝 또는 조건 분기로 처리한다.

```ts
// 금지
const name = category!.name

// 허용
const name = category?.name ?? ''
```
