# API Conventions

API 통신 계층의 구성, 타입 정의, 에러 처리에 대한 규칙.

---

## API 계층 위치

API 관련 코드는 FSD 레이어 내 `api/` 세그먼트에 위치한다.

| 역할 | 위치 |
|------|------|
| HTTP 클라이언트 설정 | `shared/api/client.ts` |
| 엔티티 조회 API 함수 | `entities/{name}/api/` |
| 기능별 뮤테이션 API 함수 | `features/{name}/api/` |
| React Query 훅 | API 함수와 같은 세그먼트 또는 `model/` |
| 쿼리 키 팩토리 | `entities/{name}/api/query-keys.ts` |

---

## HTTP 클라이언트

Axios 인스턴스를 `shared/api/client.ts`에서 단일로 생성하고 전체 프로젝트에서 공유한다. 직접 `axios.get()`을 호출하지 않는다.

```ts
// shared/api/client.ts
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10_000,
  headers: {
    'Content-Type': 'application/json',
  },
})

// 요청 인터셉터: 인증 토큰 주입
apiClient.interceptors.request.use((config) => {
  const token = authStore.getToken()
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// 응답 인터셉터: 공통 에러 처리
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      authStore.clearToken()
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

---

## API 함수 정의

### 규칙

- API 함수는 `async/await` 방식으로 작성한다.
- 반환 타입을 명시한다.
- URL은 함수 내부에 문자열로 직접 작성하지 않고 상수로 분리한다.
- 함수명은 `{동사}{대상}` 형태로 작성한다.

```ts
// entities/category/api/category-api.ts
import { apiClient } from '@/shared/api/client'
import type { Category, CategoryListParams, CategoryListResponse } from '../model/types'

const BASE_URL = '/categories'

export async function fetchCategoryList(params: CategoryListParams): Promise<CategoryListResponse> {
  const { data } = await apiClient.get<CategoryListResponse>(BASE_URL, { params })
  return data
}

export async function fetchCategoryById(id: string): Promise<Category> {
  const { data } = await apiClient.get<Category>(`${BASE_URL}/${id}`)
  return data
}

export async function createCategory(body: CreateCategoryRequest): Promise<Category> {
  const { data } = await apiClient.post<Category>(BASE_URL, body)
  return data
}

export async function updateCategory(id: string, body: UpdateCategoryRequest): Promise<Category> {
  const { data } = await apiClient.patch<Category>(`${BASE_URL}/${id}`, body)
  return data
}

export async function deleteCategory(id: string): Promise<void> {
  await apiClient.delete(`${BASE_URL}/${id}`)
}
```

---

## 요청/응답 타입 정의

### 타입 위치

요청/응답 타입은 해당 엔티티의 `model/types.ts`에 정의한다.

```ts
// entities/category/model/types.ts

// 도메인 모델
type Category = {
  id: string
  name: string
  description: string
  status: CategoryStatus
  createdAt: string
}

type CategoryStatus = 'ACTIVE' | 'INACTIVE'

// 목록 조회 파라미터
type CategoryListParams = {
  page?: number
  size?: number
  keyword?: string
  status?: CategoryStatus
}

// API 응답
type CategoryListResponse = {
  items: Category[]
  totalCount: number
  page: number
  totalPages: number
}

// 생성 요청
type CreateCategoryRequest = {
  name: string
  description?: string
}

// 수정 요청
type UpdateCategoryRequest = Partial<CreateCategoryRequest>
```

### 네이밍 규칙

| 대상 | 패턴 | 예시 |
|------|------|------|
| 도메인 모델 | `{Entity}` | `Category`, `Product` |
| 목록 조회 파라미터 | `{Entity}ListParams` | `CategoryListParams` |
| 목록 응답 | `{Entity}ListResponse` | `CategoryListResponse` |
| 생성 요청 | `Create{Entity}Request` | `CreateCategoryRequest` |
| 수정 요청 | `Update{Entity}Request` | `UpdateCategoryRequest` |
| 단건 응답 | `{Entity}` 또는 `{Entity}Detail` | `Category`, `CategoryDetail` |

---

## React Query 훅 네이밍

| 용도 | 패턴 | 예시 |
|------|------|------|
| 목록 조회 | `use{Entity}List` | `useCategoryList` |
| 단건 조회 | `use{Entity}Detail` | `useCategoryDetail` |
| 생성 | `use{Entity}Create` | `useCategoryCreate` |
| 수정 | `use{Entity}Update` | `useCategoryUpdate` |
| 삭제 | `use{Entity}Delete` | `useCategoryDelete` |

```ts
// entities/category/api/use-category-list.ts
function useCategoryList(params: CategoryListParams) {
  return useQuery({
    queryKey: categoryKeys.list(params),
    queryFn: () => fetchCategoryList(params),
  })
}

// features/category/model/use-category-delete.ts
function useCategoryDelete() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: string) => deleteCategory(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
    },
  })
}
```

---

## 에러 처리

### 에러 타입 정의

서버에서 반환하는 에러 응답 구조를 타입으로 정의한다.

```ts
// shared/api/types.ts
type ApiError = {
  status: number
  code: string
  message: string
}

function isApiError(error: unknown): error is AxiosError<ApiError> {
  return axios.isAxiosError(error) && error.response !== undefined
}
```

### 에러 처리 계층

| 레벨 | 처리 대상 | 방법 |
|------|-----------|------|
| 인터셉터 | 401 인증 만료 | 토큰 삭제 + 로그인 리다이렉트 |
| mutation `onError` | 뮤테이션 실패 | 실패 토스트 표시 |
| query `onError` / ErrorBoundary | 조회 실패 | 에러 UI + 재시도 버튼 |
| 컴포넌트 | 폼 유효성 오류 | 인라인 에러 메시지 |

```ts
// mutation 에러 처리
useMutation({
  mutationFn: createCategory,
  onError: (error) => {
    if (isApiError(error)) {
      // 서버가 내려준 메시지 사용
      toast.error(error.response.data.message)
    } else {
      toast.error('요청을 처리하지 못했습니다. 잠시 후 다시 시도해주세요.')
    }
  },
})
```

### 에러 메시지 원칙

- 서버가 사용자 친화적인 메시지를 내려주면 그대로 사용한다.
- 기술적 오류 코드(`500`, `VALIDATION_FAILED`)는 사용자에게 노출하지 않는다.
- fallback 메시지는 "요청을 처리하지 못했습니다. 잠시 후 다시 시도해주세요."로 통일한다.

---

## 공통 응답 구조

페이지네이션이 있는 목록 응답은 아래 구조를 따른다.

```ts
type PagedResponse<T> = {
  items: T[]
  totalCount: number
  page: number
  size: number
  totalPages: number
  hasNext: boolean
}
```

단건 응답은 데이터를 직접 반환한다. 불필요한 래핑 구조(`{ data: { ... } }`)는 만들지 않는다.

---

## URL 관리

API URL은 각 api 파일 상단에 상수로 선언한다. 하드코딩된 URL 문자열을 함수 곳곳에 분산하지 않는다.

```ts
// 허용
const BASE_URL = '/categories'

async function fetchCategoryList() {
  return apiClient.get(BASE_URL)
}

async function deleteCategory(id: string) {
  return apiClient.delete(`${BASE_URL}/${id}`)
}

// 금지
async function fetchCategoryList() {
  return apiClient.get('/categories')  // 문자열 직접 사용
}
```
