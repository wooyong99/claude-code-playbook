# Folder Structure

FSD 아키텍처를 기반으로 한 프로젝트 패키지 구조.

---

## 전체 구조

```
src/
├── app/                        # 진입점, 전역 설정
│   ├── providers/              # 최상위 프로바이더 (QueryClient, Router 등)
│   ├── router/                 # 라우팅 정의
│   ├── styles/                 # 글로벌 스타일
│   └── index.tsx               # App 루트 컴포넌트
│
├── pages/                      # URL 단위 페이지
│   ├── dashboard/
│   │   └── index.tsx
│   ├── login/
│   │   └── index.tsx
│   ├── tenant/
│   │   ├── register/
│   │   │   └── index.tsx
│   │   └── index.tsx
│   └── category/
│       └── index.tsx
│
├── widgets/                    # 재사용 가능한 대형 UI 블록
│   ├── header/
│   │   ├── ui/
│   │   └── index.ts
│   ├── sidebar/
│   │   ├── ui/
│   │   └── index.ts
│   └── {widget-name}/
│       ├── ui/
│       └── index.ts
│
├── features/                   # 비즈니스 기능 단위
│   ├── auth/                   # 인증/로그인
│   │   ├── ui/
│   │   ├── model/
│   │   ├── api/
│   │   └── index.ts
│   ├── category/               # 카테고리 관리
│   │   ├── ui/
│   │   ├── model/
│   │   ├── api/
│   │   └── index.ts
│   └── {feature-name}/
│       ├── ui/
│       ├── model/
│       ├── api/
│       └── index.ts
│
├── entities/                   # 도메인 핵심 데이터
│   ├── tenant/
│   │   ├── model/              # 타입, 인터페이스
│   │   ├── api/                # 조회 API
│   │   └── index.ts
│   ├── category/
│   │   ├── model/
│   │   ├── api/
│   │   └── index.ts
│   └── {entity-name}/
│       ├── model/
│       ├── api/
│       └── index.ts
│
└── shared/                     # 공통 리소스
    ├── api/                    # HTTP 클라이언트 설정
    │   └── client.ts
    ├── ui/                     # 공통 UI 컴포넌트
    │   ├── Button/
    │   ├── Input/
    │   ├── Modal/
    │   └── index.ts
    ├── lib/                    # 유틸리티 함수
    ├── hooks/                  # 공통 훅
    ├── constants/              # 상수
    ├── types/                  # 공통 타입
    └── config/                 # 환경변수, 앱 설정
```

---

## 세그먼트 역할

각 슬라이스 내부는 역할에 따라 아래 세그먼트로 구분한다.

| 세그먼트 | 역할 |
|----------|------|
| `ui/` | React 컴포넌트, 스타일 |
| `model/` | 상태 관리, 커스텀 훅, 비즈니스 로직 |
| `api/` | API 호출 함수, React Query 훅 |
| `lib/` | 슬라이스 전용 유틸리티 |
| `types/` | 슬라이스 전용 타입 정의 |
| `index.ts` | Public API — 외부에 노출할 인터페이스만 export |

세그먼트는 필요한 것만 생성한다. 모든 슬라이스가 모든 세그먼트를 가질 필요는 없다.

---

## Public API (`index.ts`) 규칙

슬라이스 외부에서 접근하는 모든 경로는 반드시 `index.ts`를 통해야 한다.

```ts
// features/auth/index.ts
export { LoginForm } from './ui/LoginForm'
export { useAuthStore } from './model/store'
```

```ts
// 허용
import { LoginForm } from '@/features/auth'

// 금지 — 내부 구현 직접 접근
import { LoginForm } from '@/features/auth/ui/LoginForm'
```

---

## Path Alias

`tsconfig.json`의 path alias를 활용해 레이어 루트를 절대 경로로 참조한다.

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

```ts
import { Button } from '@/shared/ui'
import { useCategory } from '@/entities/category'
import { CategoryForm } from '@/features/category'
```

---

## 네이밍 규칙

| 대상 | 규칙 | 예시 |
|------|------|------|
| 디렉토리 | kebab-case | `category-list/`, `tenant-register/` |
| 컴포넌트 파일 | PascalCase | `CategoryTable.tsx` |
| 훅 파일 | camelCase, `use` 접두사 | `useCategoryList.ts` |
| API 파일 | camelCase | `categoryApi.ts` |
| 타입 파일 | camelCase | `category.types.ts` |
