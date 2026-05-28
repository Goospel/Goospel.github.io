# Goospel's Learning Notes

> 작업 중 모르고 물어봐서 배운 기술 개념을 한 곳에 누적. 발표 / 면접 / Q&A 에서 본인이 직접 설명할 수 있는 수준으로 본인 이해 확립하는 것이 목적. 같은 질문 두 번 안 묻기.

## 🎯 이 사이트는

- **PKM (Personal Knowledge Management)** — 개인 학습 베이스. 프로젝트마다 새로 학습한 일반 개념을 누적
- **공개** — 같은 함정을 겪을 다른 사람도 참고 가능. 면접 / 포트폴리오 효과
- **사적 메모는 별도 관리** — 본 사이트는 일반화된 기술 개념만. 미완성 / 회고는 private vault 에 분리

## 📚 빠른 진입

<div class="grid cards" markdown>

-   :material-account-key: **인증 / 보안**

    ---

    JWT / Refresh Token / Rotation / SHA-256 / BCrypt — 보안과 UX 의 trade-off

    [→ 인증 노트](notes/auth/jwt-refresh-token.md)

-   :material-database: **데이터베이스**

    ---

    Hibernate ddl-auto 의 한계 / MySQL utf8mb4 함정 / 운영 머지 후 사후 검증

    [→ DB 노트](notes/db/hibernate-ddl-auto-limits.md)

-   :material-server-network: **운영 / CI·CD**

    ---

    dev vs main 브랜치 전략 / AWS SSM 자동화 범위

    [→ 운영 노트](notes/ops/ci-vs-cd-branch-strategy.md)

-   :material-folder-multiple: **프로젝트**

    ---

    각 프로젝트의 학습 트리거 + 외부 링크

    [→ 프로젝트 목록](projects/index.md)

</div>

## 💡 활용법

- **검색 (오른쪽 위 🔍)** — 키워드 한 단어로 찾기. "refresh", "rotation", "ddl-auto" 등
- **카테고리 네비게이션** — 인증 / DB / 운영 으로 분류. 좌측 사이드바
- **편집 링크 (각 페이지 우상단 연필 ✏️)** — GitHub 에서 직접 수정. PR 환영

## 📌 누적 정책

새 질문 / 학습이 생길 때마다 추가. 단순 정의가 아니라 **"본인이 모르고 물어봤다 → 답을 듣고 본인이 청중에게 설명 가능한 수준의 이해 확립"** 의 메타 학습 기록.

각 항목 구조:
- 한 줄 요약
- 자세한 설명 (개념 + 예시 + 표)
- 발표 시 한 줄 비유 (있으면)
- 청중 Q&A 대비 (있으면)
- 참고 / 코드 위치

## 🔗 외부

- GitHub: [Goospel](https://github.com/Goospel)
- 졸업 프로젝트: [Moodiary (Backend)](https://github.com/Goospel/hoseo-moodiary)
