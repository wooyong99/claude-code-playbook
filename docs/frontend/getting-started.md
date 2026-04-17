# Getting Started

프론트엔드 로컬 개발 환경 구성 및 실행 안내.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| 언어 | TypeScript 5.4 |
| 프레임워크 | React 18 |
| 빌드 | Vite 5 |
| 라우팅 | React Router v6 |
| 스타일 | Tailwind CSS 3 |
| HTTP | Axios |
| 린트 | ESLint |

---

## 프로젝트 구성

```
frontend/
├── admin/       # 시스템 관리자 UI
└── backoffice/  # 테넌트 운영자 UI
```

각 앱은 동일한 기술 스택과 Feature-Sliced Design(FSD) 아키텍처를 따른다.

---

## 실행

```bash
cd frontend/backoffice   # 또는 frontend/admin
npm install
npm run dev              # 개발 서버
npm run build            # 프로덕션 빌드
npm run lint             # 린트
```
