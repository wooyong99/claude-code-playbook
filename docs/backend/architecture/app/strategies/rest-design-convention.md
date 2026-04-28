# REST API 설계 컨벤션

---

## 핵심 규칙

**URL은 리소스 중심으로 설계한다. 자원은 복수형 명사 + kebab-case + `/api/v{N}/` prefix를 사용하며, 식별자는 Path parameter, 필터·정렬·페이지네이션은 Query parameter로 표현한다.**

페이지네이션은 기본적으로 **커서 기반(cursor-based)** 을 우선 고려하고, 총 페이지 수·페이지 점프가 필요한 경우에만 오프셋 기반(offset-based)을 사용한다. 응답 포맷은 두 방식 모두 `BaseResponse<T>` 안에서 `content` + `page` 구조로 일관되게 래핑한다.

> Controller / DTO / Extension 등 구현 패턴은 [api-convention.md](api-convention.md) 참고
> 예외 응답 포맷은 [exception-handling-convention.md](exception-handling-convention.md) 참고

---

## URL 네이밍

**규칙: 리소스 중심 URL — 복수형 명사 + kebab-case + `/api/v{N}/` prefix.**

### 패턴

- 도메인 개념을 **복수 명사**로 — `/api/v1/orders`, `/api/v1/fs-nodes`
- 단어 구분은 **하이픈(`-`, kebab-case)** — `/api/v1/menu-catalogs`, `/api/v1/purchase-orders`
- 버전 prefix는 `/api/v{N}` (현재 `v1`) — `/api/v1/...`
- admin 전용은 `/api/v{N}/admin/...` prefix — `/api/v1/admin/users`
- 버전은 URL path 노출 (`Accept` 헤더 버저닝 사용 안 함)
- 계층 관계는 path 중첩 — `/api/v1/orders/{orderId}/items`

### HTTP 메서드 매핑

- `GET /resources` — 목록 조회 (`GET /api/v1/orders`)
- `GET /resources/{id}` — 단건 조회 (`GET /api/v1/orders/123`)
- `POST /resources` — 생성 (`POST /api/v1/orders`)
- `PUT /resources/{id}` — 전체 수정 / 리소스 대체 (`PUT /api/v1/orders/123`)
- `PATCH /resources/{id}` — 부분 수정 (`PATCH /api/v1/orders/123`)
- `DELETE /resources/{id}` — 삭제 (`DELETE /api/v1/orders/123`)

### 동사·행위 — 원칙 금지, 예외 제한적

리소스에 자연스럽게 맵핑되지 않는 **상태 전이 / 행위**는 하위 리소스로 표현한다. URL에 동사를 직접 노출하지 않는다.

```
✅ POST /api/v1/orders/{id}/cancellation     (취소 요청을 하위 리소스로 표현)
✅ POST /api/v1/orders/{id}/confirmation
✅ POST /api/v1/fs-nodes/{id}/moves           (이동 요청)

❌ POST /api/v1/orders/{id}/cancel            (동사 노출)
❌ GET  /api/v1/orders/getDetails/{id}        (RPC 스타일)
❌ POST /api/v1/doSomething
```

### 금지 표기

```
❌ /api/v1/order_items      (snake_case)
❌ /api/v1/orderItems       (CamelCase)
❌ /api/v1/order            (단수형 — Singleton 예외 제외)
❌ /api/v1/OrdersList       (대문자 시작)
```

---

## Plural 규칙

**규칙: 리소스 경로 세그먼트는 항상 복수형을 사용한다. Singleton 리소스만 예외적으로 단수형을 허용한다.**

### 복수형 사용 — 기본

```
✅ /api/v1/orders               (주문 컬렉션)
✅ /api/v1/orders/123           (컬렉션 내 단건)
✅ /api/v1/fs-nodes
✅ /api/v1/menu-catalogs

❌ /api/v1/order
❌ /api/v1/order/123
```

컬렉션과 단건의 URL 세그먼트를 동일한 복수형으로 통일하면 클라이언트가 일관된 mental model을 유지할 수 있다.

### Singleton 리소스 — 단수형 허용

요청 컨텍스트에서 **항상 하나만 존재하는 리소스**는 단수형을 사용한다.

```
✅ GET /api/v1/me                 (현재 로그인한 사용자 본인)
✅ GET /api/v1/current-tenant     (현재 테넌트 컨텍스트)
```

### 영어 불규칙 복수 — 정확한 영어 복수형 사용

- `category` → `categories`
- `inventory` → `inventories`
- `person` → `people` (프로젝트 내 통일 필요)
- `status` → `statuses`

가능하면 **규칙적 복수(`orders`, `users`, `products`)로 표현 가능한 도메인 용어를 선택**하여 불규칙 복수 변환을 피한다.

---

## ID 위치 — Path vs Query

**규칙: 리소스 고유 식별자는 Path parameter, 선택적 필터·정렬·페이지네이션·검색은 Query parameter로 전달한다.**

### 판단 기준

- 리소스를 **고유하게 식별**하는 ID → **Path** (`/orders/{orderId}`)
- 상위 리소스 ID (계층 관계) → **Path** (`/orders/{orderId}/items/{itemId}`)
- 선택적 필터 조건 → **Query** (`/orders?status=PENDING`)
- 정렬 → **Query** (`/orders?sort=createdAt,desc`)
- 페이지네이션 → **Query** (`/orders?cursor=xxx&size=20`)
- 복수 ID 일괄 조회 → **Query** (`/orders?ids=1,2,3`)
- 검색 키워드 → **Query** (`/orders?keyword=abc`)

핵심 질문: **"이 값이 없으면 자원을 특정할 수 없는가?"**

- **예** → Path
- **아니오(컬렉션은 존재함)** → Query

### Path — 리소스 식별자

- **계층 표현**: `/api/v1/orders/{orderId}/items/{itemId}` — 상위 리소스를 path 로 명시
- **공유·북마크 가능**: 하나의 URL이 하나의 리소스를 가리킴
- **경로 자체가 의미를 가짐**: URL만으로 무엇을 요청하는지 읽힘

### Query — 필터 · 정렬 · 페이지네이션

```
GET /api/v1/orders?status=PENDING&sort=createdAt,desc&cursor=xxx&size=20
```

- `null` / 빈 값은 query string에 포함시키지 않는다.
- 배열은 **콤마 구분**을 기본으로 한다: `?ids=1,2,3` (프로젝트 내 표기 통일).

### 테넌트 ID — URL path 미노출

이 프로젝트는 멀티테넌트 구조이지만, `tenantId`는 URL path에 노출하지 않고 **요청 헤더·세션 컨텍스트**를 통해 `TenantContext`로 해석한다. URL은 테넌트 중립적으로 설계한다.

```
✅ GET /api/v1/orders                    (헤더·컨텍스트에서 tenant 해석)

❌ GET /api/v1/tenants/{slug}/orders     (tenant를 URL에 노출 — 이 프로젝트에서는 사용 안 함)
```

---

## Pagination 표준

**규칙: 페이지네이션은 기본적으로 커서 기반(cursor-based)을 우선 고려하고, 총 페이지 수·페이지 점프가 필요한 경우에만 오프셋 기반(offset-based)을 사용한다.**

### 선택 기준

- 무한 스크롤 / 최신순 피드 → **커서 기반** (데이터 추가·삭제에 안정적)
- 시간 순서가 중요한 목록 (활동 로그, 주문 이력 등) → **커서 기반**
- 관리자 테이블 — 페이지 점프 UI 필요 → 오프셋 기반
- 총 개수를 UI에 명시적으로 표시해야 함 → 오프셋 기반
- 고정된 스냅샷 목록 (검색 결과 상위 N개 등) → 오프셋 기반

둘 다 필요한 경우 엔드포인트를 분리하지 않고, **엔드포인트별로 하나의 방식만** 고정한다 (혼용 금지).

---

### 커서 기반 Pagination

**요청 파라미터**

- `cursor`: 다음 페이지 시작 커서 (opaque 문자열) — `?cursor=eyJpZCI6MTIzfQ==`
- `size`: 페이지 크기 (기본 20, 최대 100) — `?size=20`
- `sort`: 정렬 기준 — `?sort=createdAt,desc`

```
GET /api/v1/orders?cursor=eyJpZCI6MTIzfQ==&size=20&sort=createdAt,desc
```

첫 페이지 요청 시 `cursor`는 생략한다 (`GET /api/v1/orders?size=20`).

**응답 포맷**

```json
{
  "data": {
    "content": [
      { "id": 123, "name": "..." }
    ],
    "page": {
      "size": 20,
      "nextCursor": "eyJpZCI6MTIwfQ==",
      "hasNext": true
    }
  }
}
```

- `nextCursor`가 `null`이거나 `hasNext`가 `false`이면 마지막 페이지.
- `cursor`는 클라이언트가 해석하지 않는 **불투명 토큰(opaque token)** — 서버가 내부적으로 ID·timestamp 조합 등을 base64로 인코딩한다.
- 커서 기반은 **총 개수(`totalElements`)를 제공하지 않는 것**이 원칙.

---

### 오프셋 기반 Pagination

**요청 파라미터**

- `page`: 0부터 시작하는 페이지 번호 — `?page=0`
- `size`: 페이지 크기 (기본 20, 최대 100) — `?size=20`
- `sort`: 정렬 기준 — `?sort=createdAt,desc`

Spring Data의 `Pageable` 파라미터 규약과 일치한다.

```
GET /api/v1/admin/orders?page=2&size=20&sort=createdAt,desc
```

**응답 포맷**

```json
{
  "data": {
    "content": [
      { "id": 123, "name": "..." }
    ],
    "page": {
      "page": 2,
      "size": 20,
      "totalElements": 143,
      "totalPages": 8
    }
  }
}
```

---

### 공통 제약

- `size` 기본값은 **20**, 상한은 **100** (서버가 강제 clamp).
- `sort`는 `{field},{asc|desc}` 형식. 복수 정렬은 `sort=a,desc&sort=b,asc`.
- 페이지네이션 응답은 반드시 `BaseResponse<T>` 안에 `content` + `page` 구조로 래핑한다.
- 페이지네이션은 **컬렉션 엔드포인트(`GET /resources`)** 에만 적용 — 단건 조회에는 적용하지 않는다.

---

## 판단 기준 — 종합

### 행위 리소스 설계

- 리소스 CRUD에 자연스럽게 맵핑 → `POST /orders` (생성), `DELETE /orders/{id}`
- 상태 전이 — 도메인 행위 → `POST /orders/{id}/cancellation` (하위 리소스)
- 검색처럼 리소스화가 어색 → `GET /orders?keyword=...` (Query parameter)

### Path vs Query

- 값이 없으면 자원을 식별할 수 없다 → Path
- 값이 없어도 자원 컬렉션 자체는 존재한다 → Query

### 커서 vs 오프셋

- 목록 변동이 잦고 최신순 스크롤 → 커서
- 총 개수·페이지 점프가 반드시 필요 → 오프셋
- 시간순 로그·피드 → 커서
- 단순 관리자 테이블 → 오프셋

---

## 금지 사항

- URL에 동사를 노출하지 않는다 — `/orders/getDetails/{id}` 같은 RPC 스타일 금지.
- URL에 단수형을 사용하지 않는다 (Singleton 리소스 예외 제외) — `/order/{id}` 대신 `/orders/{id}`.
- URL에 snake_case·CamelCase를 사용하지 않는다 — 반드시 kebab-case.
- URL path에 `tenantId`(slug)를 노출하지 않는다 — 헤더·컨텍스트로 전달.
- 리소스 고유 식별자를 Query parameter로 받지 않는다 — 반드시 Path.
- 필터·정렬·페이지네이션을 Path로 받지 않는다 — 반드시 Query.
- 한 엔드포인트에서 **offset과 cursor 파라미터를 동시에** 받지 않는다 — 엔드포인트별로 방식 고정.
- `size` 상한을 검사하지 않고 그대로 수용하지 않는다 — 100 초과는 서버에서 clamp.
- `Accept` 헤더 버저닝을 사용하지 않는다 — `/api/v{N}/` path 버저닝만 사용.
- 페이지네이션 응답에서 `content` / `page` 외의 임의 구조를 사용하지 않는다.

---

## 체크리스트

### URL
- [ ] URL이 복수형 명사 + kebab-case + `/api/v{N}/` prefix 형식인가?
- [ ] admin 전용 URL이 `/api/v{N}/admin/...` prefix를 따르는가?
- [ ] URL에 동사·snake_case·CamelCase가 포함되지 않는가?
- [ ] HTTP 메서드가 의미에 맞게 매핑됐는가? (GET / POST / PUT / PATCH / DELETE)
- [ ] `tenantId`(slug)가 URL path에 노출되지 않는가?
- [ ] 상태 전이 행위가 하위 리소스로 표현됐는가? (`/orders/{id}/cancellation` 등)

### Plural
- [ ] 컬렉션·단건 URL 세그먼트가 모두 복수형인가?
- [ ] 불규칙 복수를 정확히 사용했는가? (`categories`, `statuses` 등)
- [ ] Singleton 리소스(`/me`, `/current-tenant`)만 단수형을 사용하는가?

### ID / Query
- [ ] 리소스 고유 식별자가 Path parameter인가?
- [ ] 필터·정렬·페이지네이션·검색이 Query parameter인가?
- [ ] 계층 관계가 path 중첩으로 표현됐는가?

### Pagination
- [ ] 엔드포인트가 커서 / 오프셋 한 방식만 사용하는가? (혼용 금지)
- [ ] `size` 기본값 20, 상한 100이 적용됐는가?
- [ ] 커서 기반 응답이 `nextCursor` / `hasNext` 메타를 포함하는가?
- [ ] 오프셋 기반 응답이 `page` / `size` / `totalElements` / `totalPages` 메타를 포함하는가?
- [ ] 페이지네이션 응답이 `BaseResponse<T>` 안에 `content` + `page` 구조로 래핑됐는가?
