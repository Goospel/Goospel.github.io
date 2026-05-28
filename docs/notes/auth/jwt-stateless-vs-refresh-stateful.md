# JWT stateless vs Refresh stateful — 본질적 차이

> **한 줄 요약**: JWT (access) 는 서명만 검증하면 되는 stateless — DB 조회 0, 빠름. 대신 한 번 발급되면 서버에서 못 막음 (만료 대기). Refresh 는 DB 에 저장된 stateful — 검증 시 DB 조회 필요하지만 서버에서 즉시 무효화 가능. 둘을 합치면 속도와 통제력의 균형.

## 비교 표

| 항목 | JWT Access | Refresh Token |
|---|---|---|
| 상태 | **Stateless** (서버에 저장 X) | **Stateful** (DB 에 저장) |
| 검증 방법 | 서명 검증 (`HMAC-SHA256`) | DB 조회 후 hash 비교 |
| 검증 속도 | μs 단위 (DB 부담 0) | ms 단위 (DB 한 번 SELECT) |
| 발급 후 서버 통제 | ❌ 불가능 — 만료 대기만 | ✅ 가능 — `revoked_at` 컬럼 update 로 즉시 무효 |
| 수명 권장 | 짧게 (1h) | 길게 (2w) |
| 사용 빈도 | 매 API 요청 | 1시간에 한 번 |

## Stateless 의 장단점

### 장점 — 속도와 확장성
- 서버는 토큰의 서명만 검증 — 자기 secret key 와 토큰의 signature 만 비교
- DB / 캐시 / 외부 서비스 호출 0
- 매 요청에 μs 단위 처리
- 수평 확장 시 서버 간 공유 상태 불필요 (각 서버가 같은 secret 만 가지면 됨)

### 단점 — 통제력 부재
- 한 번 발급된 JWT 는 만료 시각까지 무조건 유효
- 사용자가 로그아웃 해도 서버는 그 JWT 를 차단할 방법이 없음 (블랙리스트 두면 stateful 이 되어버림)
- 토큰 탈취 시 만료 대기 외에 대응 불가

→ 그래서 **JWT 수명은 짧게** (1h 등) 유지하는 게 운영 안전성과 일관됨.

## Stateful 의 장단점

### 장점 — 통제력
- DB 의 `revoked_at` 컬럼 한 줄 UPDATE 로 즉시 무효화
- 로그아웃 / 계정 침해 / 비정상 활동 감지 시 즉시 대응 가능
- 토큰의 발급 / 사용 / 무효화 이력을 추적 가능

### 단점 — 성능 비용
- 매 검증마다 DB 조회 1회
- 초당 수만 요청 처리 서비스에서는 부담
- DB 가 single point of failure

## 두 토큰을 합치는 이유

### Access 만 — 빠르지만 통제 0
```
모든 요청 마다 stateless 검증 (빠름)
탈취 시 만료 대기만 가능 (위험)
```

### Refresh 만 — 통제 가능하지만 부담
```
모든 요청 마다 DB 조회 (느림)
DB 부담 폭증
```

### 둘 다 — 양쪽의 좋은 점만
```
매 요청은 stateless access 검증 (빠름)
1시간에 한 번만 stateful refresh 갱신 (DB 부담 최소)
서버가 refresh 무효화 가능 (로그아웃 / 침해 대응)
```

## 실 운영 예시

### 일반 API 요청 — Stateless
```
GET /post/{id}
  └─ JwtAuthenticationFilter 가 서명만 검증 → DB 조회 0 → μs 처리
```

### Access 만료 후 갱신 — Stateful (1시간에 한 번)
```
POST /auth/refresh
  └─ DB 에서 refresh hash 조회 → 검증 → rotation
```

### 로그아웃 — Stateful 의 가치
```
POST /auth/logout
  └─ UPDATE refresh_token SET revoked_at = NOW()
  → 다음 refresh 시도부터 즉시 차단
  → Access 는 1시간 내 자연 차단 대기 (stateless 한계 흡수)
```

### 계정 침해 대응 — Stateful 의 가치
```
"이 사용자의 모든 refresh revoke"
  → UPDATE refresh_token SET revoked_at = NOW() WHERE user_id = :compromised
  → 사용자가 1시간 내 강제 재로그인
```

## 발표 시 한 줄

> "빠른 검증과 강한 통제력은 양립하기 어려운데, 둘로 쪼개면 양립 가능합니다. 자주 쓰이는 검증은 stateless 로 빠르게, 가끔 일어나는 갱신은 stateful 로 통제 가능하게."

## FAQ

**Q: JWT 만 쓰는 시스템도 많은데 왜 refresh 추가?**
→ JWT 만으로도 동작하지만 운영 시 "강제 로그아웃이 안 됨" 이 큰 약점. 침해 대응 / 비정상 활동 차단 / 사용자 명시 로그아웃 — 다 stateless 에선 불가.

**Q: Redis 같은 캐시에 JWT 블랙리스트를 두면?**
→ 가능하지만 그러면 매 요청마다 Redis 조회 — 사실상 stateful 이 됨. JWT 의 stateless 이점 포기. Refresh 분리 패턴이 더 깔끔.

**Q: Stateless 검증 = 진짜 DB 조회 0?**
→ 토큰 검증만 0. 비즈니스 로직이 `userId` 로 DB 조회는 별개 (사용자 정보 가져오기 등). 인증 검증 자체는 0.

## 관련

- [JWT vs Refresh Token](jwt-refresh-token.md)
- [SHA-256 token storage](sha256-token-storage.md)
- [Refresh Token Rotation](refresh-token-rotation.md)
