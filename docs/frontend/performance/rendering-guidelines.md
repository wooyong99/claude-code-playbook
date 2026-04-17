# Rendering Guidelines

불필요한 렌더링을 줄이고 성능을 유지하기 위한 렌더링 전략.

---

## 불필요한 Re-render 방지 원칙

Re-render는 상태(state)나 props가 변경될 때 발생한다. 컴포넌트가 실제로 변경된 데이터와 무관하게 렌더링되는 것을 방지하는 것이 핵심이다.

### 기본 원칙

- 상태는 그것을 실제로 사용하는 컴포넌트에 가능한 한 가깝게 위치시킨다.
- 변경 빈도가 다른 상태는 분리한다. 자주 바뀌는 상태와 거의 안 바뀌는 상태를 같은 컴포넌트에 두면 후자가 불필요하게 re-render된다.
- 컴포넌트가 사용하지 않는 props를 전달하지 않는다.
- 렌더링 중 새 객체/배열/함수를 생성하면 props가 항상 다른 참조가 되어 하위 컴포넌트가 매번 re-render된다.

```tsx
// 금지: 렌더링마다 새 객체 생성
<List style={{ marginTop: 16 }} />

// 허용: 외부에 정의하거나 useMemo 사용
const listStyle = { marginTop: 16 }
<List style={listStyle} />
```

---

## Key 안정성 유지

`key`는 React가 리스트 항목을 식별하는 데 사용한다. 불안정한 key는 불필요한 언마운트/마운트를 유발한다.

### 금지 패턴

```tsx
// 금지: 인덱스를 key로 사용 (항목 순서 변경 시 DOM 재생성)
{items.map((item, index) => <Row key={index} item={item} />)}

// 금지: 렌더링마다 새로운 값 생성
{items.map((item) => <Row key={Math.random()} item={item} />)}
```

### 허용 패턴

```tsx
// 허용: 데이터 고유 식별자 사용
{items.map((item) => <Row key={item.id} item={item} />)}
```

- 순서가 바뀌지 않고 추가/삭제도 없는 정적 목록은 인덱스 key 허용
- 정렬, 필터, 추가, 삭제가 가능한 목록은 반드시 고유 ID 사용

---

## memo / useMemo / useCallback 사용 기준

최적화 API는 측정 가능한 성능 문제가 있을 때 도입한다. 무분별한 사용은 오히려 메모리 낭비와 코드 복잡도 증가를 초래한다.

### React.memo

부모가 re-render될 때 props가 변경되지 않은 자식 컴포넌트의 re-render를 방지한다.

**적용 기준:**
- 렌더링 비용이 높은 컴포넌트 (테이블 row, 복잡한 카드)
- 부모가 자주 re-render되지만 해당 컴포넌트의 props는 변경이 적은 경우
- React DevTools Profiler로 실제 성능 병목이 확인된 경우

```tsx
// 렌더링 비용이 높은 테이블 row에 적용
const CategoryRow = React.memo(({ category, onEdit, onDelete }: Props) => {
  return <tr>...</tr>
})
```

### useMemo

계산 비용이 높은 값을 메모이제이션한다.

**적용 기준:**
- 렌더링마다 반복되는 복잡한 연산 (대용량 데이터 정렬, 필터링, 집계)
- 하위 컴포넌트에 객체/배열로 전달되는 파생값

```tsx
// 허용: 대용량 목록 필터링
const filtered = useMemo(
  () => items.filter((item) => item.name.includes(keyword)),
  [items, keyword]
)

// 금지: 단순 연산에 useMemo 적용
const count = useMemo(() => items.length, [items])  // 불필요
```

### useCallback

함수 참조를 안정화한다. `React.memo`로 감싼 컴포넌트에 콜백을 전달하거나, 해당 함수가 `useEffect`의 의존성 배열에 포함될 때 사용한다.

```tsx
// 허용: memo 컴포넌트에 전달하는 핸들러
const handleDelete = useCallback((id: string) => {
  deleteMutation.mutate(id)
}, [deleteMutation])

// 금지: memo가 아닌 일반 컴포넌트에 useCallback 적용
const handleChange = useCallback((e) => setValue(e.target.value), [])  // 불필요
```

---

## List Item 분리 기준

목록을 렌더링할 때 각 항목을 별도 컴포넌트로 분리하면 개별 항목의 변경이 목록 전체를 re-render하지 않는다.

### 분리 기준

| 조건 | 처리 방식 |
|------|-----------|
| 항목 내부에 독립적인 상태가 있는 경우 | 컴포넌트 분리 필수 |
| 항목 렌더링 로직이 복잡한 경우 (JSX 10줄 이상) | 컴포넌트 분리 권장 |
| 항목별 이벤트 핸들러가 있는 경우 | 컴포넌트 분리 후 memo 적용 검토 |
| 단순 텍스트 나열 수준인 경우 | 인라인 렌더링 허용 |

```tsx
// 권장: 항목 컴포넌트 분리
function CategoryList({ categories }: Props) {
  return (
    <ul>
      {categories.map((category) => (
        <CategoryItem key={category.id} category={category} />
      ))}
    </ul>
  )
}

const CategoryItem = React.memo(({ category }: { category: Category }) => {
  return <li>{category.name}</li>
})
```

---

## 상태 최소화

컴포넌트가 보유하는 상태의 수와 범위를 최소화한다.

### 원칙

- 다른 상태에서 **계산 가능한 값은 상태로 두지 않는다** (derived state).
- 상태는 단일 source of truth를 갖는다. 동일한 데이터를 두 곳에서 상태로 관리하지 않는다.
- 서버에서 가져온 데이터는 컴포넌트 로컬 상태에 복사하지 않는다. React Query 캐시가 단일 source of truth이다.

```tsx
// 금지: 서버 데이터를 로컬 상태에 복사
const [categories, setCategories] = useState<Category[]>([])
useEffect(() => {
  fetchCategories().then(setCategories)
}, [])

// 허용: React Query로 서버 상태 직접 관리
const { data: categories } = useQuery({ queryKey: categoryKeys.lists(), queryFn: fetchCategories })
```

---

## 상위 State 변경이 하위 전체를 흔들지 않도록 구조화

상위 컴포넌트의 상태가 변경될 때 관련 없는 하위 컴포넌트가 re-render되는 문제를 방지한다.

### 상태 격리 전략

**1. 상태를 사용하는 컴포넌트로 내려보내기 (State Colocation)**

상태를 가능한 한 실제로 사용하는 컴포넌트 가까이에 위치시킨다.

```tsx
// 금지: 상위에서 모든 상태를 관리
function Page() {
  const [isModalOpen, setIsModalOpen] = useState(false)  // 하위에서만 쓰이는 상태
  return (
    <>
      <Header />
      <CategoryTable />
      <Modal open={isModalOpen} onClose={() => setIsModalOpen(false)} />
    </>
  )
}

// 허용: 상태를 사용하는 곳으로 이동
function CategorySection() {
  const [isModalOpen, setIsModalOpen] = useState(false)
  return (
    <>
      <CategoryTable onAdd={() => setIsModalOpen(true)} />
      <Modal open={isModalOpen} onClose={() => setIsModalOpen(false)} />
    </>
  )
}
```

**2. 컨텍스트 분리**

전역 상태를 하나의 Provider에 담지 않고, 변경 빈도와 관심사에 따라 분리한다.

```tsx
// 금지: 모든 상태를 하나의 Context에
<AppProvider>  {/* 인증 + UI 상태 + 설정 모두 포함 */}

// 허용: 관심사별 분리
<AuthProvider>
  <UISettingsProvider>
    <App />
  </UISettingsProvider>
</AuthProvider>
```

---

## 서버 상태 업데이트 후 필요한 범위만 갱신

서버 데이터 변경 후 React Query 쿼리를 invalidate할 때 필요한 쿼리만 대상으로 한다.

```tsx
// 금지: 모든 쿼리 무효화
queryClient.invalidateQueries()

// 금지: 엔티티 전체 무효화 (상세와 목록이 모두 필요하지 않을 때)
queryClient.invalidateQueries({ queryKey: ['categories'] })

// 허용: 영향받는 쿼리만 대상
// 생성 → 목록만
queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })

// 수정 → 목록 + 해당 상세
queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
queryClient.invalidateQueries({ queryKey: categoryKeys.detail(id) })
```

쿼리 키는 계층적으로 설계하여 범위 지정 invalidation이 가능하도록 한다. ([state-management.md](../architecture/state-management.md) 참고)

---

## 삭제 후 전체 목록 강제 리렌더링 대신 캐시 업데이트 우선

삭제 성공 후 전체 목록을 다시 fetch하는 대신, 캐시에서 해당 항목만 제거하는 방식을 우선 검토한다.

### 캐시 직접 업데이트 (Optimistic / 성공 후)

```tsx
// 성공 후 캐시에서 직접 제거
onSuccess: (_, deletedId) => {
  queryClient.setQueryData(
    categoryKeys.lists(),
    (old: Category[] | undefined) => old?.filter((c) => c.id !== deletedId)
  )
}
```

### invalidation 방식 (단순하고 안전)

캐시 직접 조작이 복잡하거나 목록 외 다른 쿼리에도 영향이 있는 경우 invalidation을 사용한다.

```tsx
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
}
```

### 선택 기준

| 상황 | 방식 |
|------|------|
| 단순 목록에서 단일 항목 삭제 | 캐시 직접 업데이트 우선 |
| 삭제 후 집계/카운트 등 다른 쿼리에도 영향 | invalidation |
| 페이지네이션이 있는 목록 | invalidation (페이지 재계산 필요) |
| Optimistic Update 적용 | 캐시 직접 조작 + 실패 시 rollback |
