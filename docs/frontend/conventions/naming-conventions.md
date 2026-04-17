# Naming Conventions

식별자 명명 규칙. 일관된 이름은 코드 탐색과 의도 파악을 쉽게 한다.

---

## 케이스 규칙 요약

| 대상 | 케이스 | 예시 |
|------|--------|------|
| 파일명 | kebab-case | `category-list.tsx`, `use-category.ts` |
| 컴포넌트 클래스(React) | PascalCase | `CategoryTable`, `DeleteConfirmDialog` |
| 인터페이스 | PascalCase | `CategoryRowProps`, `ApiResponse` |
| 타입 | PascalCase | `Category`, `CategoryStatus` |
| 변수 | camelCase | `categoryList`, `isLoading` |
| 함수 | camelCase | `formatPrice`, `handleDelete` |
| 커스텀 훅 | camelCase + `use` 접두사 | `useCategoryList`, `useDeleteConfirm` |
| 상수 | UPPER_SNAKE_CASE | `MAX_PAGE_SIZE`, `API_BASE_URL` |
| CSS 클래스(Tailwind 외) | kebab-case | `category-table`, `form-error` |

---

## 파일명

모든 파일명은 **kebab-case**를 사용한다.

```
category-list.tsx
use-category-list.ts
category-api.ts
category.types.ts
query-keys.ts
```

컴포넌트 파일이라도 파일명 자체는 kebab-case로 작성한다. 내부에서 export되는 컴포넌트 이름은 PascalCase이다.

```tsx
// 파일명: category-table.tsx
export function CategoryTable() { ... }
```

---

## 컴포넌트명

React 컴포넌트는 **PascalCase**.

```tsx
function CategoryTable() { ... }
function DeleteConfirmDialog() { ... }
const CategoryRow = React.memo(({ ... }) => { ... })
```

컴포넌트명은 UI에서 하는 역할을 명확히 드러낸다.

| 패턴 | 예시 |
|------|------|
| 데이터 + UI 요소 | `CategoryTable`, `ProductCard` |
| 액션 + 대상 | `DeleteConfirmDialog`, `CreateCategoryModal` |
| 상태 + UI 요소 | `EmptyState`, `LoadingSkeleton` |

---

## 인터페이스 / 타입명

PascalCase. `I` 접두사나 `Type` 접미사는 사용하지 않는다.

```ts
// 금지
interface ICategoryProps { ... }
type CategoryType = { ... }

// 허용
interface CategoryRowProps { ... }
type Category = { ... }
type CategoryStatus = 'ACTIVE' | 'INACTIVE'
```

Props 타입은 `{ComponentName}Props` 패턴으로 명명한다.

```ts
interface CategoryTableProps { ... }
interface DeleteConfirmDialogProps { ... }
```

---

## 변수 / 함수명

camelCase. 이름만으로 역할과 의도를 알 수 있도록 작성한다.

### Boolean 변수

`is`, `has`, `can`, `should` 중 하나를 접두사로 사용한다.

```ts
const isLoading = true
const hasError = false
const canDelete = user.role === 'ADMIN'
const shouldRefetch = staletime > threshold
```

### 이벤트 핸들러

`handle` 접두사를 사용한다. props로 전달되는 콜백은 `on` 접두사를 사용한다.

```tsx
// 컴포넌트 내부 핸들러: handle 접두사
function handleDelete(id: string) { ... }
function handleFormSubmit(data: FormValues) { ... }

// props로 전달하는 콜백: on 접두사
interface CategoryRowProps {
  onEdit: (id: string) => void
  onDelete: (id: string) => void
}
```

### 비동기 함수

`fetch`, `get`, `create`, `update`, `delete` 등 동작을 명확히 나타내는 동사로 시작한다.

```ts
async function fetchCategoryList(params: CategoryListParams) { ... }
async function createCategory(data: CreateCategoryRequest) { ... }
async function updateCategory(id: string, data: UpdateCategoryRequest) { ... }
async function deleteCategory(id: string) { ... }
```

---

## 커스텀 훅

`use` 접두사 + 역할 설명. 반환하는 것이 무엇인지 이름에 드러낸다.

```ts
// 데이터 조회
function useCategoryList(params: CategoryListParams) { ... }
function useCategoryDetail(id: string) { ... }

// 뮤테이션
function useCategoryCreate() { ... }
function useCategoryDelete() { ... }

// UI 상태 관리
function useDeleteConfirm() { ... }
function useModalState() { ... }
```

---

## 상수

모듈 스코프에서 값이 고정된 상수는 **UPPER_SNAKE_CASE**.

```ts
const MAX_PAGE_SIZE = 20
const DEFAULT_DEBOUNCE_MS = 300
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL
const TOAST_DURATION_MS = 3000
```

컴포넌트 내부의 임시 변수는 camelCase도 허용한다.

---

## React Query 쿼리 키

쿼리 키 팩토리 객체는 camelCase + `Keys` 접미사.

```ts
export const categoryKeys = {
  all: ['categories'] as const,
  lists: () => [...categoryKeys.all, 'list'] as const,
  detail: (id: string) => [...categoryKeys.all, 'detail', id] as const,
}

export const productKeys = { ... }
export const tenantKeys = { ... }
```

---

## 디렉토리명

모든 디렉토리명은 **kebab-case**.

```
features/
  category-management/
  tenant-register/

entities/
  shop-product/
```

FSD 레이어 디렉토리(`app`, `pages`, `widgets`, `features`, `entities`, `shared`)는 소문자 그대로 사용한다.
