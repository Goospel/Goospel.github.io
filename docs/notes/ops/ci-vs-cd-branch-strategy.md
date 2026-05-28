# CI vs CD — dev/main 브랜치의 의미

> **한 줄 요약**: `dev` 머지는 **CI (테스트 + 빌드 검증)** 만 트리거. **운영 배포 (CD)** 는 `main` push 에서만 트리거. dev → main 의 release PR 머지를 따로 해야 운영 서버에 반영됨. dev 머지를 운영 반영으로 착각하면 "왜 운영에서 안 보이지?" 사고 발생.

## 흔한 흐름 — Git Flow 의 간소화 버전

```
PR → dev     →  CI 실행 (./gradlew build, 테스트)
                "코드가 빌드되는가 + 테스트가 깨지지 않는가" 만 확인
                운영에는 아무 영향 X

PR (release) → main  →  CD 실행
                        ├─ Docker Hub 에 새 이미지 push
                        ├─ AWS SSM / 그 외 도구로 운영 서버에 명령
                        └─ 컨테이너 교체 → ✅ 운영 반영 완료
```

## 왜 두 단계로 나누나

### `dev` — 통합 브랜치
- 여러 feature PR 이 머지되어 누적
- 실험적 / 위험한 변경도 일단 dev 까지는 OK (CI 통과만 하면)
- 다른 개발자와 협업 시 "내 작업 끝났다" 의 기본 단위

### `main` — 운영 브랜치
- main 의 모든 commit 은 실제로 운영 서버에 돌고 있어야 함
- main = "지금 운영에 떠 있는 상태의 진실의 원천"
- main 머지는 "운영에 보낼 시점" 결정 = 신중

### `release PR (dev → main)`
- dev 에 쌓인 여러 PR 을 묶어서 운영에 보내는 시점 결정
- 사용자가 명시적으로 타이밍 결정 (자동화 X)
- release PR body 에 "운영 머지 전 필수" 체크리스트 박는 패턴 권장

## 흔한 실수 시나리오

```
"새 기능 머지했는데 왜 운영에 안 보이지?"
→ feature → dev PR 은 머지됨 (CI 통과)
→ dev → main release PR 은 아직 안 만들거나 미머지
→ 운영 서버는 옛 main 의 코드만 돌고 있음
→ 새 기능은 dev 에만 존재
```

→ **dev 머지 = 실험 끝. 운영 반영 = 별도의 release PR 머지가 필요**.

## release PR 의 패턴

### PR body 에 박을 항목

1. **포함된 PR 목록** — 어떤 feature PR 들이 묶이는지
2. **운영 영향 분류** — 코드 변경 / 스키마 변경 / 환경변수 변경 / docs
3. **사전 체크리스트** — 운영 머지 전 사용자가 확인할 것
4. **사후 체크리스트** — 머지 후 CD 완료 후 검증할 것

### 예시

```markdown
## 포함된 PR
| PR | 운영 영향 |
|---|---|
| #63 | ⚠️ 큼 — 새 테이블 + 새 endpoint |
| #64 | 0 (docs) |

## 사전 체크리스트
- [ ] 운영 환경변수 `JWT_SECRET` 존재 재확인

## 사후 체크리스트
- [ ] CD 성공 + health check
- [ ] 새 테이블 자동 생성 확인 (`SHOW CREATE TABLE`)
- [ ] UNIQUE 인덱스 사후 검증 (`SHOW INDEX`)
- [ ] Smoke test
```

## 작은 팀 / 1인 프로젝트의 단순화

작은 팀은 `dev` 생략하고 feature 브랜치를 직접 main 으로 PR 하기도 함. 그 때는 CI 와 CD 가 같은 trigger 에 묶임:

```
PR → main → CI + CD 한 번에
```

→ 단순하지만 운영 영향 큰 PR 도 main 으로 직접 가는 위험. 팀 규모 / 운영 안정성 요구도에 따라 선택.

## 발표 시 한 줄

> "dev 머지는 코드가 빌드되고 테스트가 통과한다는 검증만 끝낸 상태이고, 운영 서버 반영은 main 으로의 release PR 머지를 따로 거쳐야 일어납니다. 이 분리가 운영 신뢰성의 핵심입니다."

## FAQ

**Q: 한 commit 으로 dev + main 둘 다 어떻게 못 하나?**
→ 가능. `git push origin dev main` 등. 다만 main 의 의미가 "운영 동기" 인데 자동으로 들어가면 위험. 의도적 release 타이밍 결정을 잃음.

**Q: dev 에 누가 직접 push 하면?**
→ Branch protection rule 로 dev / main 둘 다 직접 push 차단 권장. PR + 리뷰 + CI 통과를 거치게.

**Q: hotfix 는 어떻게?**
→ 패턴 1: `hotfix/*` 브랜치 → main 직접 PR (긴급) → 머지 후 main → dev 로 back-merge. 패턴 2: dev → main 빠르게 release. 팀 컨벤션 결정.

## 관련

- [AWS SSM 자동화 범위](aws-ssm-automation-scope.md) — CD 의 자동화 어디까지인가
