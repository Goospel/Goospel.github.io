# 학습 노트

## 🔍 카테고리

### 🔐 [인증 / 보안](auth/jwt-refresh-token.md)
JWT / Refresh Token / Rotation / SHA-256 / BCrypt / Stateless vs Stateful — 토큰 기반 인증의 trade-off

- [JWT vs Refresh Token — 왜 두 개로 쪼갰나](auth/jwt-refresh-token.md)
- [SHA-256 token storage — DB 에 hash 만 저장](auth/sha256-token-storage.md)
- [BCrypt vs SHA-256 — 같은 해시인데 왜 다른 알고리즘?](auth/bcrypt-vs-sha256.md)
- [Refresh Token Rotation — 왜 한 번 쓰면 무효화?](auth/refresh-token-rotation.md)
- [JWT stateless vs Refresh stateful](auth/jwt-stateless-vs-refresh-stateful.md)

### 🗄️ [데이터베이스](db/hibernate-ddl-auto-limits.md)
ORM / 스키마 / 인덱스 / charset

- [Hibernate `ddl-auto: update` 의 한계](db/hibernate-ddl-auto-limits.md)
- [MySQL `utf8mb4` 함정 — 이모지 INSERT 폭발](db/mysql-utf8mb4-trap.md)

### ⚙️ [백엔드 / Spring](backend/spring-async-pattern.md)
Spring 패턴 / 동시성 / 웹 표준

- [Spring 비동기 (`@EnableAsync` + `@Async`) — long-running 외부 호출 패턴](backend/spring-async-pattern.md)
- [CORS — Same-Origin Policy + Preflight + allowlist](backend/cors-fundamentals.md)

### 🚢 [운영 / CI·CD](ops/ci-vs-cd-branch-strategy.md)
브랜치 전략 / 배포 자동화 / 운영 검증

- [CI vs CD — dev/main 브랜치의 의미](ops/ci-vs-cd-branch-strategy.md)
- [AWS SSM 자동화 범위 — CD 후 수동 작업 필요한가?](ops/aws-ssm-automation-scope.md)
- [AWS SSM Run Command — outbound polling 메커니즘](ops/aws-ssm-outbound-polling.md)

## 📌 누적 정책

새 학습이 생길 때마다 추가. **"몰랐다 → 물어봤다 → 청중에게 설명 가능한 수준으로 정리"** 의 메타 학습 기록.
