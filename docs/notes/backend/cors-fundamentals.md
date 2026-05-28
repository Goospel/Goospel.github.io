# CORS — Same-Origin Policy + Preflight + allowlist vs 와일드카드

> **한 줄 요약**: **CORS** 는 브라우저의 Same-Origin Policy (다른 origin 응답 읽기 차단) 의 예외를 **서버 응답 헤더로 명시적 허용** 하는 메커니즘. 인증 헤더 / JSON body 가 들어가면 본 요청 전에 **OPTIONS preflight** 로 사전 확인. 와일드카드 `*` 은 `credentials=true` 와 충돌하므로 정확한 origin allowlist 가 사실상 유일한 안전 정책.

## Same-Origin Policy (SOP) — CORS 의 전제

브라우저의 가장 오래된 보안 규칙:
> JavaScript 가 **다른 origin 의 응답을 읽을 수 없게 차단**한다.

**Origin** = Protocol + Host + Port 세 요소가 모두 같아야 "same origin":

| URL A | URL B | Same Origin? |
|---|---|---|
| `https://example.com/page` | `https://example.com/api` | ✅ |
| `https://example.com` | `http://example.com` | ❌ (protocol 다름) |
| `https://example.com` | `https://api.example.com` | ❌ (host 다름) |
| `https://example.com:443` | `https://example.com:8080` | ❌ (port 다름) |

### SOP 가 없다면 — 악성 시나리오

```
사용자가 은행 사이트 (bank.com) 로그인 중 — 쿠키 살아있음
   │
   ↓
같은 탭으로 evil.com 방문
   │
   ↓
evil.com 의 JavaScript:
   fetch('https://bank.com/api/transfer?to=hacker&amount=1000000')
   ↑ 브라우저가 자동으로 bank.com 쿠키 첨부
   ↑ 은행 서버는 "로그인된 사용자의 정상 요청" 으로 인식
   ↑ 돈 이체 성공
```

→ SOP 가 이걸 기본적으로 차단. evil.com 의 JS 는 bank.com 의 응답을 읽을 수 없음.

## 그런데 — 정당한 cross-origin 도 많다

SPA 분리 배포 시나리오:

```
Frontend:  https://my-app-frontend.s3-website.ap-northeast-2.amazonaws.com   (S3)
Backend:   http://api.my-app.com:8080                                        (EC2 / ALB)
                                ↑ Origin 완전히 다름
```

FE 의 React 가 BE 의 API 를 호출해야 하는데 SOP 가 차단. **여기에 CORS 가 필요**.

## CORS — 서버가 "이 origin 은 OK" 라고 선언

CORS 는 HTTP 응답 헤더로 동작. 서버가 응답에 특정 헤더를 박으면, 브라우저가 "OK, 이 origin 에서 온 요청은 허용".

### 단순 요청 (Simple Request) 의 흐름

```
[브라우저 (FE)]                              [서버 (BE)]

GET /resource
Origin: https://my-app-frontend.s3-website...
                                  ─────────►  처리 후 응답

                                  ◄─────────  200 OK
                                              Access-Control-Allow-Origin: https://my-app-frontend.s3-website...
                                              [body]

브라우저: 응답 헤더의 ACAO 가 내 origin 과
일치 → JS 에 응답 전달 ✅
```

핵심 헤더:
- **요청 헤더 `Origin`** — 브라우저가 자동으로 박음. JS 가 조작 불가능.
- **응답 헤더 `Access-Control-Allow-Origin` (ACAO)** — 서버가 박음. 브라우저가 이걸 확인.

### 단순 요청의 조건 (다 충족해야 simple)

- Method: `GET` / `POST` / `HEAD` 중 하나
- Content-Type: `text/plain` / `application/x-www-form-urlencoded` / `multipart/form-data` 중 하나
- 커스텀 헤더 없음 (`Authorization` 도 없음)

→ 대부분의 REST API 는 `Authorization: Bearer ...` 또는 `Content-Type: application/json` 을 쓰므로 **simple 이 아님** → preflight 필요.

## Preflight Request (`OPTIONS`)

JWT 인증, JSON body 같은 게 들어가면 브라우저가 본 요청 전에 사전 확인:

```
[브라우저]                                    [서버]

─── ① Preflight (OPTIONS) ─────────────────►
OPTIONS /resource
Origin: https://my-app-frontend...
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization,Content-Type

                                  ◄─────────  204 No Content
                                              Access-Control-Allow-Origin: https://my-app-frontend...
                                              Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
                                              Access-Control-Allow-Headers: Authorization, Content-Type, Accept
                                              Access-Control-Allow-Credentials: true
                                              Access-Control-Max-Age: 3600

브라우저: 모두 OK → 본 요청 진행

─── ② 본 요청 (POST) ──────────────────────►
POST /resource
Origin: https://my-app-frontend...
Authorization: Bearer eyJhbGc...
Content-Type: application/json
{ ... }

                                  ◄─────────  201 Created
                                              Access-Control-Allow-Origin: https://my-app-frontend...
                                              [body]
```

→ 모든 요청이 2번 가는 게 아님. preflight 결과는 `Access-Control-Max-Age` 동안 (예: 3600초 = 1시간) 브라우저가 캐시. 그 동안 같은 endpoint 의 본 요청은 preflight 없이 바로 감.

## Spring 의 CorsConfig — 실제 셋업

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        config.setAllowedOrigins(List.of(
            "http://localhost:5173",                                 // 로컬 Vite
            "https://my-app-frontend.s3-website.ap-northeast-2.amazonaws.com"
        ));
        // ↑ 와일드카드 X — 정확한 URL allowlist

        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        // ↑ OPTIONS 필수 (preflight 자체)

        config.setAllowedHeaders(List.of(
            HttpHeaders.AUTHORIZATION,  // JWT Bearer 헤더
            HttpHeaders.CONTENT_TYPE,   // JSON body
            HttpHeaders.ACCEPT
        ));

        config.setExposedHeaders(List.of(HttpHeaders.LOCATION));
        // ↑ JS 가 response.headers.get('Location') 로 직접 읽을 수 있는 헤더 명시

        config.setAllowCredentials(true);
        // ↑ 쿠키 / Authorization 헤더를 cross-origin 으로 전송 허용

        config.setMaxAge(3600L);
        // ↑ preflight 캐시 1시간 — 같은 endpoint 매번 OPTIONS 안 보냄

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

`SecurityConfig` 가 `.cors(Customizer.withDefaults())` 만 호출하면 위 빈을 자동으로 wire.

운영 / 로컬에서 origin 이 다르므로 **환경변수로 주입** 추천:

```java
config.setAllowedOrigins(List.of(allowedOriginsEnv.split(",")));
// 예: APP_CORS_ALLOWED_ORIGINS=http://localhost:5173,https://my-app-frontend.s3-website...
```

## ⚠️ 함정 1: 와일드카드 `*` + `credentials=true` 충돌

```java
config.setAllowedOrigins(List.of("*"));   // ❌
config.setAllowCredentials(true);
```

브라우저가 거절. 이유: 와일드카드는 "아무 origin 다 OK" 인데 거기에 쿠키 / Authorization 까지 보내는 건 보안 무너짐.

→ JWT 인증을 cross-origin 으로 보내려면 **정확한 origin 명시 필수**.

## ⚠️ 함정 2: CORS 는 **브라우저** 만의 규칙

- Postman / curl / 백엔드 → 백엔드 호출은 CORS 무관 (Origin 헤더 없음 또는 무시)
- 그래서 Postman 테스트는 다 통과해도, SPA 에서 호출하면 CORS 에러 가능

→ 백엔드 단위 테스트만으로는 부족. SPA 통합 테스트가 별도로 필요한 이유.

## ⚠️ 함정 3: CORS 차단은 **서버가 차단하는 게 아님**

요청은 서버까지 도달함. 서버는 정상 처리하고 응답을 보냄. **브라우저가 응답 헤더를 보고 JS 에 전달 거부**. 즉:

- 서버 로그에는 "200 OK" 가 찍히는데
- 브라우저 콘솔에는 "CORS error" 가 뜸
- DevTools Network 탭에서는 응답이 보이지만 "blocked" 표시

→ "서버는 정상인데 왜 클라이언트가 못 받지" 사고가 여기서 발생.

## ⚠️ 함정 4: Preflight 는 인증 전에 통과해야

Spring Security 설정에서 **OPTIONS 요청은 인증 검사 X**. 안 그러면 preflight 자체가 401 로 막혀서 본 요청까지 못 감.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
    .anyRequest().authenticated()
);
```

또는 Spring Security 의 `.cors(Customizer.withDefaults())` 가 자동으로 처리해주기도 함 (버전에 따라).

## ⚠️ 함정 5: 이미 발생한 CORS error 는 "거절된 응답" — 재시도 무의미

CORS 에러는 응답에 ACAO 가 없거나 잘못된 origin 이라 브라우저가 거절한 것. 클라이언트 측 재시도 / 재인증 로직으로 풀리지 않음. **서버 설정 변경만이 답**.

## ⚠️ 함정 6: S3 / CDN 의 패턴 와일드카드 유혹

```java
config.setAllowedOriginPatterns(List.of("https://*.s3-website.*.amazonaws.com"));   // ⚠️
```

`allowedOriginPatterns` 는 Spring 이 제공하는 와일드카드 우회. 그러나:
- 의도치 않은 다른 S3 버킷도 허용
- 누군가 자기 S3 사이트에서 우리 API 호출하는 길을 염

→ 정확한 URL allowlist 만 명시하는 게 안전.

## 발표 시 한 줄

> "FE 와 BE 가 다른 origin 에 있으니 브라우저의 Same-Origin Policy 가 기본적으로 차단합니다. BE 가 Access-Control-Allow-Origin 응답 헤더로 명시적으로 허용해야 하고, JWT 인증 헤더가 있어서 본 요청 전에 OPTIONS preflight 가 먼저 갑니다. 와일드카드 대신 정확한 URL allowlist 만 쓰는데, 이게 credentials=true 와 동시 만족 가능한 유일한 방식입니다."

## FAQ

**Q: 서버에서 CORS 를 막는 게 아니라 브라우저가 막는다고요?**
→ 맞다. 요청은 서버까지 도달하고 서버는 정상 처리해서 응답을 보낸다. 단지 브라우저가 응답 헤더를 검사해서 ACAO 가 없거나 다르면 JS 의 `fetch().then()` 에 응답을 안 넘겨줄 뿐. 그래서 서버 로그에는 200 이 찍히는데 클라이언트 콘솔에는 CORS error 가 뜨는 사고가 흔하다.

**Q: Postman 으로는 되는데 브라우저에서는 안 돼요. 왜?**
→ Postman 은 브라우저가 아니라서 SOP / CORS 검사를 안 한다. Origin 헤더를 자동으로 박지도 않는다. 그래서 Postman 통과 = 백엔드 정상 동작 증명이지만, 브라우저 통과의 증명은 아니다.

**Q: 왜 와일드카드를 못 쓰나요?**
→ 와일드카드 자체는 가능하다 — `allowCredentials=false` 이면. 인증 헤더 / 쿠키를 cross-origin 으로 보내려면 `allowCredentials=true` 가 필요하고, 이 둘은 동시 사용 불가능하다는 게 CORS 표준이다. 모든 origin 에 쿠키 / Auth 를 풀어주면 보안이 무너지니까.

**Q: Preflight 가 매 요청마다 가나요? 비효율 아닌가요?**
→ 첫 요청만. 응답의 Access-Control-Max-Age 동안 (보통 1시간) 브라우저가 캐시. 같은 endpoint 의 같은 method / headers 조합이면 그 동안 본 요청만 간다.

**Q: 쿠키 기반 세션 vs Authorization Bearer 헤더 — CORS 관점에서 차이가?**
→ 쿠키는 브라우저가 자동으로 첨부 / 자동으로 보안 관리 (HttpOnly / SameSite), 다만 같은 domain 의 sub-domain 끼리만 자동 첨부 / cross-domain 은 까다로움. Authorization 헤더는 JS 가 명시적으로 첨부 — XSS 노출 위험 있지만 cross-domain 호환성 좋음. 둘 다 `credentials=true` 가 필요.

## 관련

- [JWT vs Refresh Token — stateless vs stateful](../auth/jwt-stateless-vs-refresh-stateful.md) — Authorization 헤더 사용 / 쿠키 미사용 배경
- [JWT Refresh Token 의 필요성](../auth/jwt-refresh-token.md) — `Authorization` 헤더가 CORS 의 "비단순 요청" 트리거가 됨
- [AWS SSM Run Command — outbound polling](../ops/aws-ssm-outbound-polling.md) — 같은 "네트워크 경계 / allowlist" 관점 — 거기는 인바운드 0, 여기는 origin allowlist
