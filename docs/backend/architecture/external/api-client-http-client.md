# ApiClient HTTP 클라이언트 컨벤션

[api-client-convention.md](api-client-convention.md)의 "HTTP 클라이언트 사용" 세부 구현 문서. 이 파일은 프로젝트별로 복사 후 사용하는 HTTP 클라이언트와 빈 구성 규칙을 채워 사용한다.

---

## 사용 HTTP 클라이언트

> {이 프로젝트에서 사용하는 HTTP 클라이언트를 기술한다. 예: `RestClient`, `RestTemplate`, `WebClient`}

---

## 빈 주입 방식

> {Provider 전용 빈을 어떻게 등록하고 주입받는지 기술한다. 코드 예시 포함}

```kotlin
// 예시 작성
```

> 빈 구성은 [config-convention.md](config-convention.md) 참고

---

## 금지 사항

- 메서드 내부에서 HTTP 클라이언트를 매번 재생성하지 않는다.
- > {프로젝트별 추가 금지 사항을 기술한다}

---

## 체크리스트

- [ ] HTTP 클라이언트 빈을 `@Qualifier`로 주입받았는가?
- [ ] 메서드 내부에서 클라이언트를 재생성하지 않는가?
- [ ] 빈이 [config-convention.md](config-convention.md)에 따라 올바르게 구성됐는가?
