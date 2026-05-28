# Learning Pipeline Playbook

> 새 프로젝트에서 학습을 누적하는 흐름. **본인의 PKM (Personal Knowledge Management) 시스템 운영 가이드** — 본인이 자주 참조 + 다른 사람도 같은 시스템 셋업 가능.
>
> 목적: **면접 / 본인 이해 확립** — 같은 질문 두 번 안 묻기.

## 🎯 시스템 구성

세 위치로 분담:

| 위치 | 역할 | 공개 | 마찰 |
|---|---|---|---|
| **프로젝트 `claude-docs/learning-notes.md`** | 현장 메모 — 프로젝트 맥락 포함 (PR 번호 / 코드 위치 / 트리거) | repo 공개 여부 따라 | **0** (즉시 기록) |
| **`goospel.github.io`** (이 사이트) | 일반화된 정제본 — 다른 프로젝트도 적용 가능한 형태 | 공개 | 중간 (의식적 승격) |
| **`learning-vault`** (private) | 사적 (면접 회고 / 미완성 / 학습 메타 / 독서) | private | 0 |

## 🔄 단계별 흐름

```
새 학습 발생
   │
   ├─→ 현장 메모 (프로젝트 learning-notes.md)
   │   └─ 즉시 박기. PR 번호 / 코드 위치 / 프로젝트 트리거 포함
   │
   ├─→ 사적이면 → learning-vault
   │   ├─ half-baked/   (미완성)
   │   ├─ interview-prep/ (면접 회고)
   │   ├─ reflections/  (학습 메타)
   │   └─ reading/      (도서 / 블로그)
   │
   └─→ 충분히 정제 (= 면접에서 본인 표현으로 설명 가능)
        └─ goospel.github.io 로 승격 (일반화)
           ├─ Moodiary / Foo / Bar 같은 프로젝트 맥락 제거
           ├─ docs/notes/<category>/<topic>.md
           └─ mkdocs.yml nav 등록 + git push
```

## 🏗️ 새 프로젝트 시작 시

### 필수 셋업 (Claude 가 자동 신설 제안)

1. **`claude-docs/learning-notes.md`** — 빈 골격 + 목차 + 누적 정책
2. **README / plan.md navigator 에 링크** — 발견성
3. **`.gitignore` 에 `.commit-msg-tmp`** — Claude 가 한글 commit 메시지 박을 때 쓰는 임시 파일이 add 되는 함정 차단

### 골격 템플릿

```markdown
# 학습 노트 — 작업 중 모르고 물어봐서 배운 것들

> 면접에서 본인이 직접 설명할 수 있는 수준으로 본인 이해 확립.
> 같은 질문 두 번 안 묻기.

## 📑 목차

(추가될 항목 — 비어 있어도 OK)

---

## 🔄 누적 갱신

| 일자 | 추가 항목 |
|---|---|
| YYYY-MM-DD | 초안 |
```

각 학습 항목 권장 구조:

- 한 줄 요약
- 자세한 설명 (개념 + 예시 + 표)
- 코드 / 관련 문서 위치
- 관련 노트 링크

## ⚙️ 작업 중 — 트리거

### 1. "이게 뭐지?" 형 질문이 떠오를 때

본인이 다음 형태로 물으면 → 그 순간이 learning-notes 후보:

- "이거 뭐냐"
- "왜 이런 거지"
- "기술적으로 설명해줘"
- "내가 몰라서 물어보는데"

Claude 와 작업 중이면 답변 끝에 "💡 learning-notes 추가 후보" 제안이 자동으로 옴 — OK 하면 즉시 박힘.

### 2. 1분 이상 디버깅 후

| 종류 | 위치 |
|---|---|
| **Trap (해결법)** — "이렇게 하지 마라" | `troubleshooting.md` (T-###) |
| **개념 (이해해야 할 것)** — "왜 이렇게 동작하는가" | `learning-notes.md` |

둘 다 후보가 되는 경우도 있음 — 그 때는 양쪽에 다 박음 (서로 링크).

### 3. PR 머지 직전

PR body 작성 시 둘 다 sweep:
- troubleshooting 새 항목 후보
- learning-notes 새 항목 후보

## 🚀 승격 — `goospel.github.io` 로

본인이 **"이거 학습 사이트로 옮길래"** / release 시점 발화 시:

### 5 단계

1. 프로젝트 learning-notes 의 각 항목 중 **일반화 가능한 것** 선별
   - 프로젝트 고유 ID / PR 번호 / 운영 URL 같은 맥락은 제거
   - "이 프로젝트에서 본 사고" → "일반 패턴" 으로 재서술
2. `goospel.github.io/docs/notes/<category>/<topic>.md` 신설
3. `goospel.github.io/mkdocs.yml` 의 `nav:` 에 새 항목 등록
4. `git push origin main`
5. GitHub Actions 자동 빌드 + 배포 (1-2분)

### 페이지 구조

```markdown
# 제목

> **한 줄 요약**: ...

## 문제 / 배경

## 해법 / 개념

## 예시 (코드 / 표 / 다이어그램)

## 관련
```

## 🔒 사적 분리 — `learning-vault`

| 본인 발화 | 위치 |
|---|---|
| "면접에서 못 답한 게" | `learning-vault/interview-prep/` |
| "아직 이해 안 되는" | `learning-vault/half-baked/` |
| "왜 이게 헷갈렸나" | `learning-vault/reflections/` |
| "이 책 정리" | `learning-vault/reading/` |

승격 가능 — `half-baked/` 의 항목이 충분히 정제되면 공개 사이트로 옮김.

## 🤖 Claude Code 자동화

본인은 `~/.claude/CLAUDE.md` 에 이 PKM 시스템 규칙을 박아두어, **모든 Claude Code 세션이 자동으로 따른다**. 새 프로젝트 시작 시 본인이 매번 인지하지 않아도:

- learning-notes 골격 신설 제안
- 트리거 시점에 후보 제안
- 승격 / 사적 분리 안내

## 💡 핵심 원칙

> 모든 학습은 **즉시 현장 메모 (마찰 0)** → **의식적 승격 (정제 / 일반화)** → **사적 / 공개 분리** 의 3 단계.

자주 빠지는 함정:
- ❌ "나중에 정리하지" — 안 함
- ❌ 너무 다듬으려고 함 — 흐름 멈춤
- ❌ 한 위치에 다 박음 — 사적 / 공개 분리가 안 되어 둘 다 못 씀

올바른 패턴:
- ✅ 떠오른 순간 박음 (한 줄도 OK)
- ✅ 정제는 별도 시점
- ✅ 위치 분담 명확
