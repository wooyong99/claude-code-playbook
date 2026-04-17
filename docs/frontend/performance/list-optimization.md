# List Optimization

목록 화면의 성능을 유지하기 위한 렌더링, 페이지네이션, 가상화 전략.

---

## 목록 구현 방식 선택 기준

| 데이터 규모 | 사용자 패턴 | 권장 방식 |
|-------------|-------------|-----------|
| 소규모 (~ 100건) | 전체 탐색 | 페이지네이션 또는 전체 로딩 |
| 중규모 (100 ~ 1,000건) | 페이지 단위 탐색 | 페이지네이션 |
| 대규모 (1,000건 이상) | 스크롤 탐색 | 가상 스크롤 또는 무한 스크롤 |
| 실시간 갱신 필요 | — | 짧은 staleTime + polling 또는 WebSocket |

---

## 페이지네이션

### 서버 사이드 페이지네이션 기본 원칙

- 목록 데이터는 항상 서버에서 페이지 단위로 가져온다. 전체 목록을 한 번에 fetch하지 않는다.
- 기본 페이지 크기(size)는 20으로 통일한다. 변경이 필요한 경우 사용자에게 선택권 제공.
- 페이지 파라미터는 쿼리 키에 포함하여 각 페이지를 독립적으로 캐싱한다.

```ts
const { data } = useQuery({
  queryKey: categoryKeys.list({ page, size: 20, ...filters }),
  queryFn: () => categoryApi.getList({ page, size: 20, ...filters }),
})
```

### 다음 페이지 Prefetch

현재 페이지 로딩이 완료된 후 다음 페이지를 미리 fetch하여 페이지 전환 시 로딩을 제거한다.

```ts
const { data } = useQuery({
  queryKey: categoryKeys.list({ page, size: 20 }),
  queryFn: () => categoryApi.getList({ page, size: 20 }),
})

// 다음 페이지 prefetch
useEffect(() => {
  if (data?.hasNextPage) {
    queryClient.prefetchQuery({
      queryKey: categoryKeys.list({ page: page + 1, size: 20 }),
      queryFn: () => categoryApi.getList({ page: page + 1, size: 20 }),
    })
  }
}, [data, page])
```

### URL 상태 동기화

현재 페이지, 정렬, 필터 상태는 URL 쿼리 파라미터로 관리한다.

```
/categories?page=2&sort=name&order=asc&keyword=여름
```

- 새로고침 후에도 동일한 목록 상태 복원
- 브라우저 뒤로가기 시 이전 페이지로 복귀
- URL 공유 시 동일한 화면 재현

---

## 필터 및 검색 최적화

### Debounce 적용

키보드 입력마다 API를 호출하지 않도록 검색 입력에 debounce를 적용한다.

```ts
const [keyword, setKeyword] = useState('')
const debouncedKeyword = useDebounce(keyword, 300)  // 300ms

const { data } = useQuery({
  queryKey: categoryKeys.list({ keyword: debouncedKeyword }),
  queryFn: () => categoryApi.getList({ keyword: debouncedKeyword }),
})
```

- debounce 지연 기본값: **300ms**
- 즉각적인 반응이 필요한 경우(드롭다운 선택 등): debounce 없이 처리

### 필터 변경 시 페이지 초기화

필터나 검색어가 변경되면 페이지를 1로 초기화한다.

```ts
const handleFilterChange = (newFilter: Filter) => {
  setFilter(newFilter)
  setPage(1)  // 페이지 초기화
}
```

---

## 테이블 렌더링 최적화

### Row 컴포넌트 분리 + memo

테이블 row는 별도 컴포넌트로 분리하고 `React.memo`를 적용한다. 단일 row 변경 시 전체 테이블이 re-render되는 것을 방지한다.

```tsx
function CategoryTable({ categories }: { categories: Category[] }) {
  return (
    <table>
      <tbody>
        {categories.map((category) => (
          <CategoryRow key={category.id} category={category} />
        ))}
      </tbody>
    </table>
  )
}

const CategoryRow = React.memo(({ category }: { category: Category }) => {
  return (
    <tr>
      <td>{category.name}</td>
      <td>{category.status}</td>
    </tr>
  )
})
```

### 이벤트 핸들러 안정화

Row에 전달되는 콜백 함수는 `useCallback`으로 참조를 안정화한다.

```tsx
const handleEdit = useCallback((id: string) => {
  openEditModal(id)
}, [])  // 의존성이 없으면 빈 배열

const handleDelete = useCallback((id: string) => {
  setDeleteTargetId(id)
  openConfirmDialog()
}, [])
```

### 컬럼 정렬 상태 관리

정렬 상태는 URL 쿼리 파라미터로 관리한다. 정렬 변경 시 페이지를 1로 초기화한다.

```ts
const handleSort = (column: string) => {
  const newOrder = sort === column && order === 'asc' ? 'desc' : 'asc'
  setSort(column)
  setOrder(newOrder)
  setPage(1)
}
```

---

## 가상 스크롤 (Virtual Scroll)

1,000건 이상의 데이터를 한 화면에 렌더링해야 하는 경우 가상 스크롤을 적용한다.

### 적용 기준

| 조건 | 처리 방식 |
|------|-----------|
| 1,000건 미만 | 페이지네이션 |
| 1,000건 이상 + 스크롤 탐색 필요 | 가상 스크롤 (TanStack Virtual 등) |
| 무한 스크롤 + 대용량 | 가상 스크롤 + 무한 쿼리 |

가상 스크롤은 DOM에 실제로 렌더링되는 항목 수를 뷰포트에 보이는 수만큼으로 제한하여 메모리와 렌더링 비용을 줄인다.

```ts
import { useVirtualizer } from '@tanstack/react-virtual'

const rowVirtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 48,  // 예상 row 높이 (px)
})
```

---

## 무한 스크롤

페이지네이션보다 연속적인 탐색 경험이 필요한 경우 무한 스크롤을 적용한다.

### useInfiniteQuery 사용

```ts
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: categoryKeys.list({ size: 20 }),
  queryFn: ({ pageParam = 1 }) => categoryApi.getList({ page: pageParam, size: 20 }),
  getNextPageParam: (lastPage) => lastPage.hasNext ? lastPage.page + 1 : undefined,
  initialPageParam: 1,
})
```

### Intersection Observer로 자동 로딩

```tsx
const observerRef = useRef<HTMLDivElement>(null)

useEffect(() => {
  const observer = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
        fetchNextPage()
      }
    },
    { threshold: 0.1 }
  )
  if (observerRef.current) observer.observe(observerRef.current)
  return () => observer.disconnect()
}, [hasNextPage, isFetchingNextPage, fetchNextPage])
```

### 페이지네이션 vs 무한 스크롤 선택 기준

| 기준 | 페이지네이션 | 무한 스크롤 |
|------|-------------|-------------|
| 특정 페이지로 바로 이동 필요 | 적합 | 부적합 |
| URL로 목록 상태 공유 | 적합 | 부적합 |
| 연속적인 탐색 경험 | 보통 | 적합 |
| Backoffice 관리 테이블 | **권장** | 비권장 |

Backoffice 특성상 특정 항목을 찾아 관리하는 패턴이 많으므로 **페이지네이션을 기본**으로 한다.
무한 스크롤은 로그, 활동 내역 등 시간순 탐색 목록에 한해 사용한다.

---

## 목록 Skeleton 기준

목록 로딩 중 스켈레톤 표시 방식은 실제 컨텐츠 구조를 반영한다.

```tsx
function CategoryTableSkeleton() {
  return (
    <table>
      <tbody>
        {Array.from({ length: 10 }).map((_, i) => (
          <tr key={i}>
            <td><div className="h-4 w-32 bg-gray-200 rounded animate-pulse" /></td>
            <td><div className="h-4 w-16 bg-gray-200 rounded animate-pulse" /></td>
            <td><div className="h-4 w-24 bg-gray-200 rounded animate-pulse" /></td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

- 스켈레톤 행 수는 기본 페이지 크기(20)와 동일하게 설정
- 테이블 헤더(컬럼명)는 로딩 중에도 유지
- 필터/정렬 변경 시에도 동일한 스켈레톤 적용
