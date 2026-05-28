# SHA-256 token storage — DB 에 hash 만 저장하는 이유

> **한 줄 요약**: Refresh token 의 raw 값을 DB 에 저장하면 DB 가 유출됐을 때 모든 사용자 세션이 즉시 탈취된다. SHA-256 해시만 저장하면 DB 가 털려도 raw 를 복원할 수 없어 안전.

## SHA-256 이 뭔가

**한 방향 해시 함수** — 임의 길이 입력을 고정 64자 hex 로 변환.

```
SHA-256("hello") → "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
SHA-256("hellp") → "8b2c86ea4c01f12b29c6fafa39d3edc1a1ec3df00b6cf6e1a9b1456b3a4e6c6c"
                  └─ 한 글자만 달라도 완전히 다른 출력 (avalanche effect)
```

3가지 핵심 특성:

| 특성 | 의미 |
|---|---|
| **Deterministic** | 같은 입력 → 항상 같은 출력 (검증 가능) |
| **Avalanche** | 입력 1비트 변경 → 출력 절반 비트 변경 (예측 불가) |
| **One-way** | 출력에서 입력 복원 수학적으로 불가능 |

## 저장 방식

```
서버: refresh token 발급
  raw = "Xy7-aBcDeF..." (32바이트 secure random, URL-safe base64)
        │
        ├─→ 클라이언트에게 raw 그대로 응답 (단 한 번)
        │
        └─→ DB 에 SHA-256(raw) 저장 = "a3f5c8...(64자 hex)"
```

## 왜 raw 가 아닌 hash?

**시나리오 — DB 가 유출**:

| 저장 방식 | DB 유출 시 결과 |
|---|---|
| raw token | 공격자가 모든 사용자의 raw token 즉시 사용 가능 → 전체 세션 탈취 |
| **SHA-256 hash** | 공격자가 hash 만 봄 → raw 복원 불가 → 세션 사용 불가 |

## 검증 시 어떻게 비교하나?

```
1. 클라이언트가 raw "Xy7-aBcDeF..." 를 헤더로 전송
2. 서버가 SHA-256 한 번 계산 → "a3f5c8..."
3. DB 의 hash "a3f5c8..." 와 비교 → 일치 → 유효!
```

서버는 매번 해시 한 번만 더 계산. raw 는 절대 DB 에 안 남음.

## 같은 원리 — 비밀번호도

비밀번호도 평문 저장 X, 해시만 저장. 다만 알고리즘은 다름:

| 입력 | 해시 알고리즘 | 이유 |
|---|---|---|
| Refresh token (서버 발급, 32바이트 random) | **SHA-256** (빠름) | 입력이 이미 강해서 brute force 불가 |
| 비밀번호 (사용자 입력, 약함) | **BCrypt** (느림) | 사용자가 약한 비밀번호 쓸 수 있어서 brute force 차단 필요 |

→ [BCrypt vs SHA-256 비교](bcrypt-vs-sha256.md)

## 왜 MD5 / SHA-1 이 아닌 SHA-256?

| 알고리즘 | 출력 크기 | 충돌 공격 (collision) |
|---|---|---|
| MD5 | 128-bit | 2004년 발견 — deprecated |
| SHA-1 | 160-bit | 2017년 발견 (SHAttered) — deprecated |
| **SHA-256** | **256-bit** | **현재까지 없음 — 표준** |

→ 현재 보안 표준은 SHA-2 family (SHA-256 / SHA-512). SHA-3 도 있지만 호환성과 성숙도로 SHA-256 이 사실상 표준.

## 발표 시 한 줄 비유

> "원본 갱신권은 사용자에게만 주고, DB 에는 사본 (한 번 가공된 해시) 만 보관. 사본을 봐도 원본을 만들 수 없으니, DB 가 털려도 갱신권은 안전."

## 관련

- [JWT vs Refresh Token](jwt-refresh-token.md)
- [BCrypt vs SHA-256](bcrypt-vs-sha256.md)
- [Refresh Token Rotation](refresh-token-rotation.md)
