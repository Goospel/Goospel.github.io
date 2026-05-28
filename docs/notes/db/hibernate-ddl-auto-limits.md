# Hibernate `ddl-auto: update` 의 한계

> **한 줄 요약**: Hibernate 의 `ddl-auto: update` 는 새 테이블 / 새 컬럼은 자동 추가하지만 **인덱스 / UNIQUE 제약은 추가를 보장하지 않는다**. 운영 머지 직후 `SHOW INDEX` 로 사후 검증 필수. 누락 시 수동 DDL 적용.

## Hibernate ddl-auto 모드 비교

| 모드 | 동작 | 사용 시점 |
|---|---|---|
| `create` | 부팅 시 모든 테이블 DROP → 새로 CREATE. **데이터 다 날아감** | 로컬 / 테스트 |
| `create-drop` | 부팅 시 CREATE, 종료 시 DROP | 통합 테스트 |
| `update` | 누락 컬럼/테이블 추가. 데이터 보존. **인덱스/UNIQUE 보장 X** | 스키마 미확정 단계 |
| `validate` | 스키마와 엔티티 비교만. 다르면 부팅 실패 | 안정화 단계 (Flyway/Liquibase 와 조합) |
| `none` | 아무것도 안 함 | 명시적 마이그레이션 도구 사용 시 |

## 왜 update 모드를 쓰는가

프로젝트 초기엔 스키마가 자주 바뀌어서 수동 DDL 작성 부담이 큼. update 모드면 새 엔티티 필드 추가 시 Hibernate 가 부팅 시 자동으로 `ALTER TABLE ADD COLUMN` 해줌. 데이터도 보존.

## update 의 한계 — 인덱스 / UNIQUE 가 안 들어갈 수 있다

```java
@Table(uniqueConstraints = @UniqueConstraint(
    name = "uk_some_unique_col",
    columnNames = "some_col"
))
```

이 코드를 새 엔티티에 추가하고 부팅해도 — **운영 DB 에 UNIQUE 인덱스가 안 들어갈 수 있음**.

### 왜?

- ddl-auto: update 는 **컬럼 추가** 는 보장하지만 **제약 변경** 은 환경/Hibernate 버전에 따라 다름
- 특히 기존 테이블에 사후 추가된 UNIQUE 는 누락 잦음
- 새로 생성되는 테이블의 UNIQUE 도 보장이 약함

## 운영 머지 후 사후 검증 흐름

```bash
# 1. 테이블 자동 생성 확인
mysql -e "SHOW CREATE TABLE my_table\G"

# 2. 인덱스 / UNIQUE 확인
mysql -e "SHOW INDEX FROM my_table;"
```

`SHOW INDEX` 결과에서 핵심 컬럼 — `Key_name` 와 **`Non_unique`**:
- `Non_unique = 0` → UNIQUE 인덱스 ✅
- `Non_unique = 1` → 일반 인덱스 (UNIQUE 아님)
- 행 자체가 없으면 → 인덱스 자체 누락

### UNIQUE 가 빠져 있으면 수동 적용

```sql
CREATE UNIQUE INDEX uk_some_unique_col ON my_table (some_col);
```

### 일반 인덱스 누락 시

```sql
CREATE INDEX idx_my_table_user_id ON my_table (user_id);
```

## 영구 해소 — Flyway 또는 Liquibase

`ddl-auto: validate` 로 전환하고 마이그레이션 도구 (Flyway / Liquibase) 로 DDL 을 명시적 SQL 파일로 관리.

### Flyway 도입 후의 모습

```
src/main/resources/db/migration/
├── V1__baseline.sql           # 현재 운영 스키마 베이스라인
├── V2__add_refresh_token.sql  # 다음 마이그레이션
└── V3__add_indexes.sql        # 인덱스 명시
```

- 부팅 시 `flyway_schema_history` 테이블 확인 → 안 적용된 마이그레이션 실행
- 모든 인덱스 / UNIQUE / 제약이 SQL 에 명시되어 누락 없음

## 발표 시 한 줄

> "스키마 미확정 단계라 자동 마이그레이션을 받고 있는데, 인덱스 보장이 안 돼서 운영 머지 직후 `SHOW INDEX` 로 사후 검증을 합니다. Flyway 도입 시 영구 해소됩니다."

## FAQ

**Q: 그럼 처음부터 Flyway 쓰는 게 낫지 않나?**
→ 초기에 스키마 자주 바뀌면 매번 마이그레이션 SQL 작성이 부담. 안정화될 때까지 update 로 빠른 반복 → 안정화 후 Flyway 전환이 실용적 흐름.

**Q: PostgreSQL 도 같은 함정?**
→ 비슷. ORM 의 자동 DDL 은 어느 DB 든 인덱스/제약 보장이 약하다. 운영 사후 검증이 안전.

**Q: `validate` 모드는 언제 켜나?**
→ 스키마가 안정됐고 Flyway/Liquibase 같은 명시 도구가 운영 DDL 을 관리할 때. `validate` 가 켜져 있으면 엔티티와 DB 가 불일치하면 부팅 실패 — 안전망 역할.

## 관련

- [MySQL utf8mb4 함정](mysql-utf8mb4-trap.md) — DB charset 도 운영 머지 전 사전 검증 필요
