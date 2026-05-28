# MySQL `utf8mb4` 함정 — 이모지 INSERT 폭발

> **한 줄 요약**: MySQL 의 `utf8` (= `utf8mb3`) 는 최대 3바이트만 저장. 이모지 (😊) 는 4바이트라 INSERT 시 폭발. 이모지 저장 의도가 있으면 DB 와 모든 테이블이 `utf8mb4` 여야 함.

## MySQL 의 charset 함정

| 이름 | 실제 의미 | 이모지 가능? |
|---|---|---|
| `utf8` (옛 이름) | 최대 3바이트 = `utf8mb3` 의 alias | ❌ |
| `utf8mb3` | 최대 3바이트 | ❌ |
| `utf8mb4` | **최대 4바이트 (진짜 UTF-8)** | ✅ |

→ MySQL 의 `utf8` 은 **표준 UTF-8 의 부분집합** (3바이트 BMP — Basic Multilingual Plane 만). 진짜 UTF-8 = `utf8mb4`.

### 왜 MySQL 만 이렇게 헷갈리는가

MySQL 이 UTF-8 을 처음 도입했던 시절 (2003년경) 표준 UTF-8 의 4바이트 영역이 거의 안 쓰여서 3바이트 한정으로 구현. 이후 표준이 진화하면서 이모지 / 한자 확장 영역이 4바이트로 들어왔는데, MySQL 은 호환성 때문에 `utf8` 이름을 그대로 두고 `utf8mb4` 를 추가.

→ 새 프로젝트는 무조건 `utf8mb4` 써야 함.

## 사고 시나리오 — 이모지 INSERT 폭발

운영 RDS 가 `utf8mb3` 면:

```sql
INSERT INTO ai_response (emoji) VALUES ('😊');
-- ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x98\x8A' for column 'emoji'
```

원인: `😊` (U+1F60A) = 4바이트 UTF-8 = `F0 9F 98 8A`. utf8mb3 컬럼이 첫 바이트 `F0` 를 보고 "내가 처리 못 하는 4바이트 시퀀스" 라며 거부.

### 운영 영향

- 비즈니스 로직 (예: AI 응답 저장) 전체 실패
- 사용자에게는 "AI 응답이 안 와요" 같은 표면 증상
- 디버깅이 어려움 — 일부 입력에만 폭발 (영문/한글은 3바이트 안이라 OK)

## 사전 검증 — 운영 머지 전

```bash
mysql -e "
  SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
  FROM information_schema.SCHEMATA
  WHERE SCHEMA_NAME='my_database';

  SELECT TABLE_NAME, TABLE_COLLATION
  FROM information_schema.TABLES
  WHERE TABLE_SCHEMA='my_database';
"
```

### 정상 기준

- 스키마 `charset = utf8mb4`, `collation = utf8mb4_*`
- 모든 테이블의 `TABLE_COLLATION` 이 `utf8mb4_*` 로 시작

### 어디든 `utf8mb3` 발견 시 사전 마이그레이션

```sql
-- 1. DB 자체
ALTER DATABASE my_database CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 2. 각 테이블
ALTER TABLE post CONVERT TO CHARACTER SET utf8mb4;
ALTER TABLE comment CONVERT TO CHARACTER SET utf8mb4;
-- ... 모든 테이블 반복
```

## Collation 선택 — `utf8mb4_unicode_ci` vs `utf8mb4_general_ci`

| Collation | 정렬 정확도 | 속도 |
|---|---|---|
| `utf8mb4_unicode_ci` | Unicode 표준 따라 정확 (한글 / 일본어 / 다국어 OK) | 약간 느림 |
| `utf8mb4_general_ci` | 단순 비교 (영문 위주, 다국어 부정확) | 빠름 |
| `utf8mb4_0900_ai_ci` | MySQL 8.0+ 권장. Unicode 9.0 기반, 더 정확 + 빠름 | 빠름 |

→ MySQL 8 이상이면 `utf8mb4_0900_ai_ci`, 5.7 이하 호환이면 `utf8mb4_unicode_ci`. `general_ci` 는 옛 default 라 명시적으로 피하는 게 안전.

## 클라이언트 / 드라이버 측 설정

DB / 테이블이 utf8mb4 여도 클라이언트 connection 이 utf8mb3 면 깨질 수 있음.

### Spring Boot JDBC URL

```yaml
spring:
  datasource:
    url: jdbc:mysql://host:3306/db?useUnicode=true&characterEncoding=utf8mb4
```

또는 connection init query:
```sql
SET NAMES utf8mb4;
```

## 발표 시 한 줄

> "이모지 저장이 핵심 기능이라 DB 가 utf8mb4 여야 하는데, MySQL 의 `utf8` 은 사실 3바이트만 저장하는 함정이라 4바이트인 이모지가 폭발합니다. 운영 머지 전 `information_schema.SCHEMATA` 와 `TABLES` 의 collation 둘 다 확인했습니다."

## FAQ

**Q: PostgreSQL 도 같은 함정?**
→ 없음. PostgreSQL 의 UTF-8 은 처음부터 4바이트까지 표준 지원. `utf8mb4` 같은 별도 이름 없음. MySQL 만의 역사적 함정.

**Q: 이모지 안 쓰면 utf8mb3 도 괜찮나?**
→ 일부 한자 확장 영역도 4바이트라 영향 받음. 그리고 사용자 입력의 미래 변화 (이모지 갑자기 들어옴) 에 무방비. 새 프로젝트는 utf8mb4 default 로.

**Q: 컬럼이 utf8mb4 인데 인덱스 크기 초과 에러?**
→ MySQL 5.7 이전엔 utf8mb4 컬럼 인덱스 크기 제한이 있었음 (`Specified key was too long; max key length is 767 bytes`). `innodb_large_prefix = ON` + `ROW_FORMAT = DYNAMIC` 설정으로 해결. 5.7+ 는 기본 OK.

## 관련

- [Hibernate ddl-auto 한계](hibernate-ddl-auto-limits.md) — 스키마 검증 패턴
