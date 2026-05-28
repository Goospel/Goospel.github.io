# Moodiary — Backend

> 호서대학교 컴퓨터공학부 졸업프로젝트 (2026). 일기 + AI 감성 분석 + 월별 무드 캘린더 플랫폼의 백엔드 API 서버.

## 🔗 외부 링크

- **GitHub 레포**: [Goospel/hoseo-moodiary](https://github.com/Goospel/hoseo-moodiary)
- **운영 Swagger UI**: [http://15.165.95.129:8080/swagger-ui/index.html](http://15.165.95.129:8080/swagger-ui/index.html)
- **랜딩 페이지 + 발표 슬라이드**: [hoseo-moodiary GitHub Pages](https://goospel.github.io/hoseo-moodiary/)
- **OpenAPI Spec**: [/v3/api-docs](http://15.165.95.129:8080/v3/api-docs)

## 🎯 무엇을 하는 프로젝트인가

사용자가 일기를 작성하면 AI 가 **공감 메시지 + 그 날의 기분 이모지**를 생성. 그 이모지는 월별 캘린더 화면에 매핑되어 한 달치 감정을 한눈에 보여줌.

3인 분업 — Backend (이 레포), Frontend (React+Vite, S3), AI 추론 서버 (별도 EC2).

## 🏗️ 핵심 스택 (Backend)

| 분류 | 기술 |
|---|---|
| 언어 / 프레임워크 | Java 25, Spring Boot 4.0.6, Spring Security 6, Spring Data JPA, QueryDSL 5 |
| 데이터 | MySQL 9.x (AWS RDS, utf8mb4), Hibernate 7.2 |
| 인증 | JWT HS256 (1h) + **Refresh Token Rotation** (2w, SHA-256 hex) + BCrypt |
| 비동기 | `@EnableAsync` + DB 상태머신 (`AiResponse PENDING→DONE/FAILED`) |
| 운영 | Docker + AWS EC2 + Elastic IP + RDS + GitHub Actions + AWS SSM `send-command` |

## 💡 이 프로젝트에서 시작된 학습 노트

| 노트 | 트리거 |
|---|---|
| [JWT vs Refresh Token](../notes/auth/jwt-refresh-token.md) | PR 10 (Refresh Token 도입) 후 "왜 두 개로 쪼갰는지" 설명 못 함 |
| [SHA-256 token storage](../notes/auth/sha256-token-storage.md) | "DB 에 SHA-256 으로 저장한다는데 SHA-256 이 뭐지?" |
| [BCrypt vs SHA-256](../notes/auth/bcrypt-vs-sha256.md) | "비밀번호는 BCrypt, refresh 는 SHA-256 — 왜 다른가?" |
| [Refresh Token Rotation](../notes/auth/refresh-token-rotation.md) | Rotation 의 보안 효과 이해 |
| [JWT stateless vs Refresh stateful](../notes/auth/jwt-stateless-vs-refresh-stateful.md) | 두 토큰의 본질적 차이 |
| [Hibernate ddl-auto 한계](../notes/db/hibernate-ddl-auto-limits.md) | PR 4-pre 머지 후 UNIQUE 인덱스 사후 검증 필요 |
| [MySQL utf8mb4 함정](../notes/db/mysql-utf8mb4-trap.md) | 이모지 저장 전 운영 RDS charset 사전 검증 |
| [CI vs CD 브랜치 전략](../notes/ops/ci-vs-cd-branch-strategy.md) | "ai_response 테이블 없는데?" — dev 머지 ≠ 운영 반영 |
| [AWS SSM 자동화 범위](../notes/ops/aws-ssm-automation-scope.md) | "CD 끝났는데 EC2 에서 docker pull 다시 해야 하나?" |

## 🛠️ 핵심 학습 정리

### 5층 결함 대사건 (T-019)
운영 CORS 검증에서 401 표면 → 실제는 컨테이너 restart loop. SSM agent 죽음 + CD silent fail + springdoc 비호환 + Post→User FK orphan + `@SpringBootTest @Disabled` 안전망 부재의 5층 동시 노출. **표면 증상이 모든 층을 가렸다.**

방어선 박힌 결과:
1. `ApplicationContext` 부팅 테스트 — 빈 wiring 결함을 빌드 단계에서 검증
2. CD `wait command-executed` + health check — silent fail 차단
3. Image SHA 태그 — 응급 롤백 옵션
4. `docker inspect .Created` 휴리스틱 — 운영 진단 첫 명령

### 핵심 정책 결정
- **UUID v4 PK** — 분산 환경 가정 + ID 추측 방지
- **`application.yaml` 추적** — 모든 값 `${ENV_VAR:기본값}` 플레이스홀더. 평문 비밀값 금지
- **`ddl-auto: update`** (안정화 시 `validate`) — 스키마 미확정 단계 손DDL 부담 제거
- **CORS 명시 origin allowlist** — 와일드카드 금지 (`allowCredentials=true` 와 충돌)
- **Refresh Token Rotation** — 탈취 노출 창 최소화

자세한 내용은 [GitHub 레포의 `claude-docs/`](https://github.com/Goospel/hoseo-moodiary/tree/dev/claude-docs) 에 정리됨.
