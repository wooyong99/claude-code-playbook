# Security Policy

백엔드 보안 관련 정책.

---

## 비밀번호

- 비밀번호는 **해시만** 도메인 계층에 진입한다.
- 평문 비밀번호는 API DTO 경계에서 종료한다 (도메인/영속 계층으로 흘러들지 않는다).

---

## 인증 컨텍스트 분리

- 모듈별로 인증 컨텍스트를 분리한다.
  - `:app:storefront`, `:app:backoffice` → `Account`
  - `:app:admin` → `SystemAdmin`
- 동일 사용자 테이블을 공유하지 않는다.

---

## 민감 정보 취급

- `.env`, credentials, 비밀키 등 민감 필드는 **커밋 금지**.
- 민감 정보가 포함된 파일은 `.gitignore`에 사전 등록한다.
