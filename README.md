# Goospel's Learning Notes

> 개인 학습 베이스 — 작업 중 모르고 물어봐서 배운 기술 개념을 누적. 발표 / 면접 / Q&A 대비.

🔗 **사이트**: [https://goospel.github.io/](https://goospel.github.io/)

## 빌더

[MkDocs Material](https://squidfunk.github.io/mkdocs-material/) — markdown 기반 정적 사이트.

## 구조

```
docs/
├── index.md            # 홈
├── about.md            # 본인 소개
├── projects/           # 프로젝트별 정보
└── notes/              # 카테고리별 학습 노트
    ├── auth/           # 인증 / 보안
    ├── db/             # 데이터베이스
    └── ops/            # 운영 / CI·CD
```

## 로컬 미리보기

```bash
pip install -r requirements.txt
mkdocs serve  # http://localhost:8000
```

## 배포

`main` push → GitHub Actions (`.github/workflows/deploy.yml`) 가 자동 빌드 + Pages 배포.

## 누적 정책

새 학습이 생길 때마다 `docs/notes/<category>/<topic>.md` 추가 + `mkdocs.yml` 의 `nav:` 에 등록.

각 노트 권장 구조:
- 한 줄 요약
- 자세한 설명 (개념 + 예시 + 표)
- 발표 시 한 줄 비유 (있으면)
- 청중 Q&A 대비 (있으면)
- 관련 노트 링크

## 기여

오타 / 잘못된 설명 발견 시 PR / Issue 환영.

## 라이선스

학습 노트 — 누구나 자유롭게 참고 / 인용 가능. 출처 표시 부탁.
