# Component Conventions

컴포넌트 작성 방식과 책임 범위에 대한 규칙.

---

## 컴포넌트는 흐름만 담당

컴포넌트는 **UI 조합과 렌더링 흐름**만 책임진다. 비즈니스 로직, API 호출, 데이터 변환은 컴포넌트 외부로 분리한다.

```tsx
// 금지: 컴포넌트 안에 비즈니스 로직 혼재
function CategoryPage() {
  const [categories, setCategories] = useState([])

  useEffect(() => {
    fetch('/api/categories')
      .then((res) => res.json())
      .then((data) => setCategories(data.filter((c) => c.status === 'ACTIVE')))
  }, [])

  const handleDelete = async (id: string) => {
    await fetch(`/api/categories/${id}`, { method: 'DELETE' })
    setCategories((prev) => prev.filter((c) => c.id !== id))
  }

  return <CategoryTable categories={categories} onDelete={handleDelete} />
}

// 허용: 흐름만 담당
function CategoryPage() {
  const { categories, isLoading } = useCategoryList()
  const { mutate: deleteCategory } = useCategoryDelete()

  if (isLoading) return <CategoryTableSkeleton />

  return <CategoryTable categories={categories} onDelete={deleteCategory} />
}
```

---

## 비즈니스 로직은 별도로 분리

데이터 조회, 변환, 뮤테이션은 커스텀 훅으로 분리한다.

| 역할 | 위치 |
|------|------|
| 서버 데이터 조회 | `entities/{name}/api/` 또는 `features/{name}/model/` |
| 데이터 변환/파생 | `features/{name}/model/` 또는 `shared/lib/` |
| 뮤테이션 (생성/수정/삭제) | `features/{name}/model/` |
| UI 상태 관리 | `features/{name}/model/` |

```ts
// features/category/model/use-category-delete.ts
function useCategoryDelete() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: string) => categoryApi.delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: categoryKeys.lists() })
    },
  })
}
```

---

## Props는 명시적으로 설계

Props는 컴포넌트가 실제로 사용하는 값만 받는다. 과도하게 범용적인 props나 사용하지 않는 props를 전달하지 않는다.

### 명시적 props 선언

```tsx
// 금지: 객체 전체를 그대로 전달
<CategoryRow category={category} />  // category 객체 내 10개 필드 중 2개만 사용

// 허용: 실제 필요한 값만 전달
<CategoryRow id={category.id} name={category.name} />

// 또는 타입을 좁혀서 전달
type CategoryRowProps = Pick<Category, 'id' | 'name' | 'status'>
```

### Props 설계 원칙

- boolean props는 명시적인 이름을 사용한다. `flag`, `value` 같은 모호한 이름은 금지.
- 이벤트 핸들러 props는 `on` 접두사를 사용한다.
- optional props는 기본값(defaultProps 또는 기본 매개변수)을 지정한다.

```tsx
interface CategoryRowProps {
  id: string
  name: string
  isActive: boolean           // ✓ 명확한 boolean 이름
  onEdit: (id: string) => void
  onDelete: (id: string) => void
  className?: string          // optional은 기본값 지정
}

function CategoryRow({ id, name, isActive, onEdit, onDelete, className = '' }: CategoryRowProps) {
  ...
}
```

---

## 공통 컴포넌트 무분별 사용 금지

`shared/ui`에 있는 공통 컴포넌트는 기능에 종속되지 않는 범용 컴포넌트만 포함한다. 특정 기능이나 도메인에 종속된 로직을 공통 컴포넌트에 추가하지 않는다.

```
shared/ui/
  Button/       ← 범용 버튼 (스타일, 크기, 로딩 상태만)
  Input/        ← 범용 입력 (스타일만)
  Modal/        ← 범용 모달 (열림/닫힘 제어만)
  Table/        ← 범용 테이블 레이아웃 (데이터 없음)
```

```tsx
// 금지: 공통 컴포넌트에 도메인 로직 추가
function Modal({ onClose, onDelete }: { onClose: () => void; onDelete?: () => void }) {
  // 삭제 기능이 모달에 종속됨
}

// 허용: 공통 모달은 열림/닫힘만 담당
function Modal({ open, onClose, children }: ModalProps) { ... }

// 삭제 확인 모달은 features에서 조합
function DeleteConfirmModal({ targetName, onConfirm, onClose }: DeleteConfirmModalProps) {
  return (
    <Modal open onClose={onClose}>
      <p>{targetName}을 삭제하시겠습니까?</p>
      <Button variant="danger" onClick={onConfirm}>삭제</Button>
    </Modal>
  )
}
```

---

## feature 전용 컴포넌트와 shared 컴포넌트 구분

| 구분 | 위치 | 기준 |
|------|------|------|
| shared 컴포넌트 | `shared/ui/` | 도메인/기능에 무관하게 재사용 가능 |
| feature 전용 컴포넌트 | `features/{name}/ui/` | 특정 기능 내에서만 사용 |
| entity UI 컴포넌트 | `entities/{name}/ui/` | 도메인 데이터 표시 전용 (상태 없음) |

새 컴포넌트를 만들 때 아래 질문으로 위치를 결정한다.

1. **"이 컴포넌트가 다른 도메인에서도 그대로 쓸 수 있는가?"** → Yes: `shared/ui`
2. **"이 컴포넌트가 특정 기능의 흐름에 종속되는가?"** → Yes: `features/{name}/ui`
3. **"이 컴포넌트가 데이터 표시만 하고 뮤테이션이 없는가?"** → Yes: `entities/{name}/ui`

---

## children 남용 금지

`children`은 컴포넌트 내부 구조를 외부에서 완전히 제어해야 할 때만 사용한다. 단순히 텍스트나 단일 요소를 전달하기 위해 children을 쓰지 않는다.

```tsx
// 금지: 단순 텍스트 전달에 children 사용
<Button>저장</Button>  // 내부에서 children을 그대로 렌더링만 함

// 허용: 명시적 prop으로 전달
<Button label="저장" />

// 허용: children이 적합한 경우 — 레이아웃/래퍼 컴포넌트
<Card>
  <CardHeader title="카테고리 목록" />
  <CardBody>
    <CategoryTable />
  </CardBody>
</Card>

// 허용: Modal처럼 내부 구조를 외부에서 자유롭게 구성할 때
<Modal open={isOpen} onClose={handleClose}>
  <DeleteConfirmContent name={targetName} onConfirm={handleDelete} />
</Modal>
```

children을 사용한다면 타입을 명시한다.

```tsx
interface CardProps {
  children: React.ReactNode  // 범용
  // 또는
  children: React.ReactElement  // 단일 element만 허용
}
```

---

## 너무 큰 컴포넌트 분리 기준 (단일 책임 원칙)

하나의 컴포넌트가 하나의 책임만 갖도록 유지한다.

### 분리 신호

아래 조건 중 하나라도 해당하면 컴포넌트 분리를 검토한다.

| 신호 | 기준 |
|------|------|
| JSX 길이 | 80줄 초과 |
| `useState` 수 | 5개 이상 |
| 독립적인 UI 섹션 | 2개 이상 |
| 재사용 가능한 블록 | 다른 곳에서도 필요한 UI |
| 조건부 렌더링 복잡도 | 3단계 이상 중첩 |

### 분리 방법

```tsx
// 분리 전: 너무 많은 책임
function CategoryPage() {
  // 필터 상태
  const [keyword, setKeyword] = useState('')
  const [status, setStatus] = useState<CategoryStatus>('ACTIVE')

  // 모달 상태
  const [isCreateModalOpen, setIsCreateModalOpen] = useState(false)
  const [isDeleteModalOpen, setIsDeleteModalOpen] = useState(false)
  const [deleteTargetId, setDeleteTargetId] = useState<string | null>(null)

  // 데이터
  const { data } = useCategoryList({ keyword, status })

  return (
    <div>
      {/* 필터 UI */}
      {/* 테이블 */}
      {/* 생성 모달 */}
      {/* 삭제 confirm 모달 */}
    </div>
  )
}

// 분리 후: 단일 책임
function CategoryPage() {
  return (
    <div>
      <CategoryFilterBar />       {/* 필터 상태 + UI */}
      <CategoryTableSection />    {/* 테이블 + 모달 조합 */}
    </div>
  )
}
```

---

## 컴포넌트 파일 내부 구조

파일 내 코드 배치 순서를 아래로 통일한다.

```tsx
// 1. import
import { ... } from '...'

// 2. 타입/인터페이스 정의
interface CategoryRowProps { ... }

// 3. 상수 (컴포넌트 외부에서 정의)
const COLUMN_WIDTHS = { name: 200, status: 100 }

// 4. 컴포넌트 정의
function CategoryRow({ ... }: CategoryRowProps) {
  // 4-1. 훅 (useState, useQuery 등)
  // 4-2. 파생 값 (useMemo, 계산된 변수)
  // 4-3. 이벤트 핸들러 (useCallback 포함)
  // 4-4. 조기 반환 (로딩, 에러, 빈 상태)
  // 4-5. return JSX

  return (...)
}

// 5. export
export { CategoryRow }
export type { CategoryRowProps }
```
