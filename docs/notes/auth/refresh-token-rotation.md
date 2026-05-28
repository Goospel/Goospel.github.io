# Refresh Token Rotation — 왜 한 번 쓰면 무효화하나

> **한 줄 요약**: Rotation 없이 refresh 가 2주간 계속 유효하면 한 번 탈취된 refresh 로 2주 동안 무한 access 갱신 가능. Rotation = "한 번 쓰면 즉시 새 걸로 교체" → 훔친 refresh 는 단 한 번만 유효.

## Rotation 없는 시나리오 (취약)

```
1. 사용자 로그인 → refresh A 발급 (2주 유효)
2. 공격자가 refresh A 탈취 (XSS / 패킷 캡처 / DB 유출 등)
3. 공격자가 refresh A 로 access 계속 갱신 → 2주간 무한 접근
   사용자는 자기 토큰이 도둑맞은 줄도 모름
```

## Rotation 있는 시나리오 (강함)

```
1. 사용자 로그인 → refresh A 발급
2. 공격자가 refresh A 탈취
3. 사용자가 1시간 후 access 갱신 시도 → refresh A 로 호출
   서버: "refresh A revoke + refresh B 발급" ← Rotation!
4. 공격자가 refresh A 로 갱신 시도 → "이미 revoke" → 401 거절
```

## 효과 — 두 가지

### 1. 탈취 노출 창 최소화
- 훔친 refresh 는 **단 1회만 유효**
- 정당한 사용자가 먼저 쓰면 훔친 건 즉시 폐기

### 2. 보안 모니터링 신호
- 사용자가 자기가 갱신 안 했는데 refresh 가 무효 응답을 받으면 = **"누가 내 토큰을 먼저 썼다"** 신호
- 전체 계정 강제 로그아웃 / 알림 가능 (선택적 후속 구현)
- "토큰 도난" 을 **사용자가 알 수 있게** 됨

## 구현 — DB 컬럼 + 트랜잭션

```
refresh_token 테이블
  refresh_token_id    UUID PK
  user_id             FK
  refresh_token_hash  SHA-256 hex (UNIQUE)
  expires_at          만료 시각
  revoked_at          무효화 시각 (null = 유효, 값 있음 = revoke 됨)
```

### `/auth/refresh` 트랜잭션 흐름

```sql
BEGIN;
  -- 1. 받은 raw 를 hash 해서 DB 에서 찾기
  SELECT * FROM refresh_token
    WHERE refresh_token_hash = SHA256(:raw)
      AND revoked_at IS NULL
      AND expires_at > NOW()
    FOR UPDATE;  -- 비관 락 (race condition 방지)

  -- 2. 기존 토큰 즉시 revoke
  UPDATE refresh_token SET revoked_at = NOW() WHERE refresh_token_id = :found_id;

  -- 3. 새 raw + new hash 생성, 저장
  INSERT INTO refresh_token (...) VALUES (..., SHA256(:new_raw), ...);
COMMIT;

-- 클라이언트에 새 access + 새 raw refresh 응답
```

## Race Condition 우려

**시나리오**: 사용자가 동시에 두 곳에서 refresh 호출 (모바일 + 태블릿 같이)

```
요청 A → 트랜잭션 시작 → SELECT FOR UPDATE → revoke + 새 refresh B
요청 B → 트랜잭션 시작 → SELECT FOR UPDATE (대기)
                       → A 끝난 후 보면 revoked_at 박혀 있음 → "이미 revoke" 응답
```

→ 한 쪽이 먼저 처리됨. 나머지는 "이미 revoke 됨" 응답을 받고 클라이언트가 재로그인 흐름으로 fallback.

### 실 운영의 추가 옵션

- **Grace window** (예: 5초) — 방금 revoke 된 토큰을 5초간은 허용. 모바일/태블릿 동시 갱신의 일시적 race 흡수
- **Token family ID** — 같은 로그인 세션에서 발급된 모든 refresh 를 family 로 묶음. 한 번에 revoke 가능 (계정 침해 대응)

## 발표 시 한 줄

> "갱신권을 한 번 쓰면 즉시 새 갱신권으로 교체되어서, 누가 훔쳐도 정당한 주인이 먼저 쓰면 훔친 건 무용지물이 됩니다."

## FAQ

**Q: 사용자가 빨리 두 번 클릭하면?**
→ 첫 클릭이 트랜잭션 끝나기 전에 두 번째 클릭이 도착하면 두 번째는 "비관 락 대기" → 첫 클릭 끝나면 "이미 revoke" 응답. 클라이언트 재시도 / 재로그인.

**Q: Rotation 이 필수인가?**
→ 보안 강화이지만 필수는 아님. 트래픽이 많고 race condition 부담이 크면 생략 가능. 단 탈취 노출 창이 길어지는 trade-off 인지.

**Q: 모든 refresh 가 어차피 2주 후 만료되는데 굳이 rotation?**
→ "2주 무방비" vs "1회 무효" 의 차이. 한 번 탈취 시 노출이 14일 → 1회로 수 천 배 단축.

## 관련

- [JWT vs Refresh Token](jwt-refresh-token.md)
- [SHA-256 token storage](sha256-token-storage.md)
- [JWT stateless vs Refresh stateful](jwt-stateless-vs-refresh-stateful.md)
