# Caching Strategy

React Query를 기반으로 한 서버 상태 캐싱 전략.

---

## 캐싱 계층

| 계층 | 도구 | 역할 |
|------|------|------|
| 서버 상태 캐시 | React Query | API 응답 데이터 캐싱, 동기화 |
| 전역 클라이언트 상태 | Zustand | 인증 정보 등 순수 클라이언트 상태 |
| 브라우저 스토리지 | localStorage | 인증 토큰 등 영속 데이터 |

서버에서 가져온 데이터는 React Query 캐시가 단일 source of truth이다. 컴포넌트 로컬 상태에 복사하지 않는다.

---

## staleTime / gcTime 기준

### 개념

- `staleTime`: 데이터를 "신선한(fresh)" 상태로 유지하는 시간. 이 시간 동안은 캐시를 그대로 사용하고 background refetch를 하지 않는다.
- `gcTime` (구 `cacheTime`): 사용되지 않는 캐시를 메모리에서 제거하기 전까지 유지하는 시간.

### 기준

| 데이터 유형 | staleTime | gcTime | 근거 |
|-------------|-----------|--------|------|
| 목록 (자주 변경) | 30초 | 5분 | 다른 사용자 수정 반영 필요 |
| 목록 (변경 드문) | 5분 | 10분 | 카테고리, 설정 등 |
| 상세 페이지 | 1분 | 5분 | 편집 전 최신 상태 확인 |
| 코드성 데이터 | 10분 | 30분 | 거의 변경되지 않는 참조 데이터 |
| 사용자 프로필 | 5분 | 10분 | 세션 중 변경 가능성 낮음 |

```ts
// 기본값 설정 (QueryClient)
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30 * 1000,       // 30초
      gcTime: 5 * 60 * 1000,      // 5분
      retry: 1,
      refetchOnWindowFocus: false, // Backoffice 특성상 비활성화
    },
  },
})
```

`refetchOnWindowFocus`는 Backoffice 특성상 기본 비활성화한다. 탭 전환 시마다 API 호출이 발생하면 운영 환경에서 불필요한 서버 부하가 생긴다.

---

## 쿼리 키 설계

쿼리 키는 계층적으로 설계하여 범위 invalidation이 가능하도록 한다.

### 팩토리 패턴

각 엔티티마다 쿼리 키 팩토리를 `entities/{name}/api/` 세그먼트에서 정의한다.

```ts
// entities/category/api/queryKeys.ts
export const categoryKeys = {
  all: ['categories'] as const,
  lists: () => [...categoryKeys.all, 'list'] as const,
  list: (params: CategoryListParams) => [...categoryKeys.lists(), params] as const,
  details: () => [...categoryKeys.all, 'detail'] as const,
  detail: (id: string) => [...categoryKeys.details(), id] as const,
}
```

### 사용 예

```ts
// 특정 목록 조회
useQuery({ queryKey: categoryKeys.list({ page: 1, size: 20 }) })

// 특정 상세 조회
useQuery({ queryKey: categoryKeys.detail(id) })

// 목록 전체 invalidation (파라미터 무관)
queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })

// 엔티티 전체 invalidation
queryClient.invalidateQueries({ queryKey: categoryKeys.all })
```

---

## Prefetch 전략

사용자가 다음에 볼 가능성이 높은 데이터를 미리 fetch하여 로딩 지연을 줄인다.

### 적용 시점

| 상황 | 방법 |
|------|------|
| 목록 항목에 hover 시 상세 예상 | hover 이벤트에서 `prefetchQuery` |
| 페이지네이션의 다음 페이지 | 현재 페이지 로딩 완료 후 `prefetchQuery` |
| 사이드바 메뉴 hover 시 해당 페이지 데이터 | 메뉴 hover에서 `prefetchQuery` |

```ts
// 목록 row hover 시 상세 prefetch
const handleRowMouseEnter = (id: string) => {
  queryClient.prefetchQuery({
    queryKey: categoryKeys.detail(id),
    queryFn: () => categoryApi.getById(id),
    staleTime: 30 * 1000,
  })
}
```

Prefetch는 선택적 최적화이므로, 측정 결과 로딩 지연이 체감되는 경우에만 도입한다.

---

## Invalidation 범위 기준

서버 상태 변경 후 캐시를 무효화할 때 영향받는 범위만 대상으로 한다.

| 액션 | Invalidation 대상 |
|------|------------------|
| 생성 (POST) | 해당 엔티티 목록 (`lists()`) |
| 수정 (PUT/PATCH) | 해당 엔티티 목록 + 해당 상세 (`detail(id)`) |
| 삭제 (DELETE) | 해당 엔티티 목록 (`lists()`) |
| 순서 변경 | 해당 엔티티 목록 (`lists()`) |

```ts
// 생성 성공
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
}

// 수정 성공
onSuccess: (_, { id }) => {
  queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
  queryClient.invalidateQueries({ queryKey: categoryKeys.detail(id) })
}
```

---

## 캐시 직접 업데이트 (setQueryData)

API 재호출 없이 캐시를 직접 수정하여 즉각적인 UI 반응을 제공한다.

### 사용 기준

- 단순 목록에서 단일 항목 삭제/수정
- Optimistic Update 구현 시
- 캐시 조작 로직이 단순하고 오류 가능성이 낮은 경우

```ts
// 수정 성공 후 캐시 직접 업데이트
onSuccess: (updatedCategory) => {
  // 목록 캐시 업데이트
  queryClient.setQueryData(
    categoryKeys.lists(),
    (old: Category[] | undefined) =>
      old?.map((c) => (c.id === updatedCategory.id ? updatedCategory : c))
  )
  // 상세 캐시 업데이트
  queryClient.setQueryData(categoryKeys.detail(updatedCategory.id), updatedCategory)
}
```

캐시 조작이 복잡해지거나 다른 쿼리에도 영향이 있으면 `invalidateQueries`로 전환한다.

---

## 에러 처리 및 재시도

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // 4xx 오류는 재시도하지 않음 (클라이언트 오류)
        if (error instanceof ApiError && error.status < 500) return false
        // 5xx 오류는 1회 재시도
        return failureCount < 1
      },
    },
    mutations: {
      retry: 0,  // 뮤테이션은 재시도하지 않음 (중복 제출 방지)
    },
  },
})
```

- 조회(query)는 네트워크 오류에 한해 1회 재시도 허용
- 변경(mutation)은 재시도하지 않는다 (동일 요청 중복 실행 방지)
- 401 응답은 재시도 없이 로그아웃 처리
