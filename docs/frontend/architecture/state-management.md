# State Management

상태를 어디에, 어떻게 두는지에 대한 기준을 정의한다.

---

## 상태 분류

| 분류 | 설명 | 예시 |
|------|------|------|
| 서버 상태 | 서버에서 fetch한 데이터. 캐싱, 동기화 필요 | 상품 목록, 카테고리 |
| 전역 클라이언트 상태 | 여러 컴포넌트가 공유해야 하는 앱 상태 | 인증 정보, 사이드바 열림 상태 |
| 로컬 UI 상태 | 단일 컴포넌트 내에서만 유효한 상태 | 모달 열림 여부, input 포커스 |
| 폼 상태 | 사용자 입력값 및 검증 상태 | 등록 폼 필드, 에러 메시지 |

---

## 전역 상태 사용 기준

전역 상태는 아래 조건을 **모두** 충족할 때만 도입한다.

1. **두 개 이상의 슬라이스(features, widgets, pages)** 에서 동일한 값을 공유해야 할 때
2. **props로 전달하기 어려운 구조**일 때 (컴포넌트 트리 깊이가 깊거나 관계가 없는 컴포넌트 간 공유)
3. **서버 상태가 아닌** 순수 클라이언트 상태일 때 (서버 상태는 React Query로 관리)

### 전역 상태로 관리하는 항목

- 인증 정보 (액세스 토큰, 사용자 정보)
- UI 전역 설정 (사이드바 열림 상태, 테마)

### 전역 상태로 관리하지 않는 항목

- 서버에서 가져온 데이터 → React Query로 관리
- 단일 컴포넌트 내 UI 상태 → `useState`로 관리
- 폼 입력값 → 로컬 상태 또는 폼 라이브러리로 관리

---

## Props Drilling 방지 기준

props drilling은 **2단계까지 허용**한다. 3단계 이상으로 전달해야 하는 경우 아래 방법 중 하나를 선택한다.

### 판단 기준

```
Page → Widget → Feature      ← 2단계, 허용
Page → Widget → Feature → UI ← 3단계, 개선 필요
```

### 해결 방법

| 상황 | 해결 방법 |
|------|-----------|
| 서버 데이터를 여러 하위 컴포넌트가 사용 | React Query 훅을 하위 컴포넌트에서 직접 호출 |
| 컴포넌트 합성으로 해결 가능한 경우 | children 또는 render props 패턴 |
| 동일 레이어 내 여러 컴포넌트가 상태 공유 | 공통 부모로 상태 끌어올리기 |
| 슬라이스 간 공유가 필요한 UI 상태 | 전역 클라이언트 상태 (Zustand 등) |

> 서버 상태를 전달하기 위해 props drilling을 하는 경우, 하위 컴포넌트에서 React Query 훅을 직접 호출하는 것이 우선이다.

---

## Derived State 원칙

이미 존재하는 상태에서 파생할 수 있는 값은 별도 상태로 저장하지 않는다.

### 금지 패턴

```ts
// 금지: items에서 파생 가능한 값을 별도 상태로 저장
const [items, setItems] = useState<Item[]>([])
const [count, setCount] = useState(0)   // items.length와 동일

useEffect(() => {
  setCount(items.length)
}, [items])
```

### 올바른 패턴

```ts
// 허용: 렌더링 시점에 계산
const [items, setItems] = useState<Item[]>([])
const count = items.length  // 파생값은 변수로 선언

// 계산 비용이 높은 경우 useMemo 사용
const expensiveValue = useMemo(() => computeExpensive(items), [items])
```

### 판단 기준

- 다른 상태나 props로부터 **계산 가능한 값**이라면 `useState` 사용 금지
- 계산 비용이 낮으면 단순 변수로 선언
- 계산 비용이 높으면 `useMemo`로 메모이제이션
- `useEffect`로 상태를 동기화하는 패턴은 derived state의 신호 — 제거를 검토한다

---

## 폼 상태 처리 기준

### 단순 폼 (필드 3개 이하, 단순 검증)

`useState`로 직접 관리한다.

```ts
const [email, setEmail] = useState('')
const [password, setPassword] = useState('')
```

### 복잡한 폼 (필드 다수, 복잡한 검증, 다단계)

폼 라이브러리를 도입한다. (예: React Hook Form)

```ts
const { register, handleSubmit, formState: { errors } } = useForm<FormValues>({
  resolver: zodResolver(schema),
})
```

### 기준 요약

| 조건 | 방법 |
|------|------|
| 필드 1~3개, 검증 단순 | `useState` |
| 필드 4개 이상 또는 복잡한 검증 | React Hook Form + Zod |
| 다단계 폼, 조건부 필드 | React Hook Form |

### 공통 원칙

- 폼 상태는 해당 feature 또는 컴포넌트 내에서만 관리한다. 전역 상태로 올리지 않는다.
- 제출 후 폼 초기화는 라이브러리의 `reset()` 또는 상태 초기값으로 처리한다.
- 서버 에러는 폼 상태와 별도로 관리한다 (`setError` 또는 로컬 상태).

---

## 캐시 Invalidation 기준

React Query를 기준으로 한다.

### Invalidation 트리거 시점

| 액션 | Invalidation 대상 |
|------|------------------|
| 데이터 생성 (POST) | 해당 엔티티 목록 쿼리 |
| 데이터 수정 (PUT/PATCH) | 해당 엔티티 목록 + 상세 쿼리 |
| 데이터 삭제 (DELETE) | 해당 엔티티 목록 쿼리 |

### 쿼리 키 설계

쿼리 키는 계층적으로 설계하여 범위 invalidation이 가능하도록 한다.

```ts
// 쿼리 키 팩토리
export const categoryKeys = {
  all: ['categories'] as const,
  lists: () => [...categoryKeys.all, 'list'] as const,
  list: (params: CategoryListParams) => [...categoryKeys.lists(), params] as const,
  detail: (id: string) => [...categoryKeys.all, 'detail', id] as const,
}
```

```ts
// 목록 전체 invalidation
queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })

// 특정 항목 invalidation
queryClient.invalidateQueries({ queryKey: categoryKeys.detail(id) })

// 엔티티 전체 invalidation
queryClient.invalidateQueries({ queryKey: categoryKeys.all })
```

### Optimistic Update 기준

- 단순 목록 변경(삭제, 상태 토글)은 optimistic update를 적용해 UX를 개선한다.
- 복잡한 연산(결제, 재고 차감 등)은 서버 응답 후 invalidation 방식을 사용한다.

---

## 삭제/수정 후 재조회 방식 기준

### 기본 원칙

삭제 또는 수정 성공 후 **화면에 반영하는 기준은 Invalidation + 자동 재조회**이다. 로컬 상태를 직접 수정하는 방식은 서버 상태와 불일치를 야기할 수 있으므로 사용하지 않는다.

### 삭제 처리

```ts
const deleteMutation = useMutation({
  mutationFn: (id: string) => categoryApi.delete(id),
  onSuccess: () => {
    // 목록 쿼리 무효화 → 자동 재조회
    queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
  },
})
```

### 수정 처리

```ts
const updateMutation = useMutation({
  mutationFn: (data: UpdateCategoryRequest) => categoryApi.update(data),
  onSuccess: (_, variables) => {
    // 목록 + 상세 모두 무효화
    queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
    queryClient.invalidateQueries({ queryKey: categoryKeys.detail(variables.id) })
  },
})
```

### 예외: 즉각적인 UI 반응이 필요한 경우

사용자 경험상 서버 응답 대기 시간이 체감될 수 있는 경우, optimistic update를 적용하되 실패 시 롤백을 반드시 구현한다.

```ts
const deleteMutation = useMutation({
  mutationFn: categoryApi.delete,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: categoryKeys.lists() })
    const previous = queryClient.getQueryData(categoryKeys.lists())
    queryClient.setQueryData(categoryKeys.lists(), (old) =>
      old?.filter((item) => item.id !== id)
    )
    return { previous }
  },
  onError: (_, __, context) => {
    // 실패 시 이전 상태로 복구
    queryClient.setQueryData(categoryKeys.lists(), context?.previous)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
  },
})
```
