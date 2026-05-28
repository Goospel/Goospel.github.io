# AWS SSM 자동화 범위 — CD 후 수동 작업 필요한가?

> **한 줄 요약**: 잘 설정된 CD 는 SSM 으로 EC2 안에서 `docker-compose pull` + `docker-compose up -d` + health check 까지 자동 실행. CD 초록불 = EC2 컨테이너가 새 이미지로 재시작 완료 + health check 통과 상태. 수동 SSH + docker pull 불필요.

## AWS SSM 이 뭔가

**Systems Manager** — AWS 의 운영 자동화 도구 모음. 그 중 `send-command` 가 핵심.

### SSM send-command 의 의미

```
GitHub Actions (CD 워크플로우)
  └─ aws ssm send-command --instance-ids i-xxx --document AWS-RunShellScript \
       --parameters commands='[
         "cd ~/my-app",
         "docker-compose pull",
         "docker-compose up -d"
       ]'
  ↓
  AWS SSM API
  ↓
  EC2 의 SSM Agent (백그라운드 데몬)
  ↓
  EC2 안에서 명령 실행 (SSH 없이)
```

→ **SSH 키 관리 불필요**. GitHub Actions 에 SSH private key 박지 않아도 됨. AWS OIDC 로 자격증명 받고 SSM 으로 명령 전달.

## CD 가 자동으로 하는 일 — 5단계

잘 설정된 CD 의 흐름:

```
1. GitHub Actions OIDC 로 AWS 자격증명 획득 (IAM Role 사용, 임시 자격증명)
2. Docker Hub 또는 ECR 에 새 이미지 push (latest + :SHA 두 태그)
3. AWS SSM send-command 로 EC2 에 명령 발사:
     "cd ~/my-app && docker-compose pull && docker-compose up -d"
4. SSM wait command-executed — 명령 실행 완료까지 대기 (timeout 설정)
5. health check — 애플리케이션 endpoint 가 200 응답할 때까지 polling
```

여기까지 **모두 자동**. 사용자가 EC2 SSH 로 들어가서 수동으로 할 일 없음.

## ⚠️ 흔한 함정 — silent fail 가능성

### 함정 1: SSM send-command 가 enqueue 만 되고 실패해도 성공 응답

- SSM API 는 명령을 큐에 넣자마자 "성공" 반환 가능
- EC2 안에서 명령이 실제로 실패해도 GitHub Actions 가 초록불
- 결과: CD 는 성공인데 운영은 옛 이미지

**방어선**: `aws ssm wait command-executed` 명령으로 실행 완료까지 대기 + 결과 확인

```yaml
- name: Wait for SSM command
  run: |
    aws ssm wait command-executed \
      --command-id ${{ steps.run.outputs.command-id }} \
      --instance-id ${{ vars.EC2_INSTANCE_ID }}
```

### 함정 2: 컨테이너는 떴지만 애플리케이션 부팅 실패

- `docker-compose up -d` 는 컨테이너만 띄움. 그 안의 애플리케이션 부팅 실패 (DB 연결 실패 / 설정 오류 등) 는 별개
- 컨테이너가 restart loop 빠지면 CD 가 그 사실 모름

**방어선**: health check polling

```yaml
- name: Health check
  run: |
    for i in {1..30}; do
      if curl -fs http://${{ vars.EC2_HOST }}:8080/health; then
        echo "Healthy!" && exit 0
      fi
      sleep 10
    done
    echo "Health check failed" && exit 1
```

### 함정 3: docker-compose pull 이 캐시된 옛 이미지 사용

- 같은 태그 (예: `latest`) 의 이미지가 이미 로컬에 있으면 docker 가 pull 생략
- `pull_policy: always` 또는 명시적 image tag (`:SHA`) 사용으로 회피

```yaml
# compose.yaml
services:
  app:
    image: myregistry/app:latest
    pull_policy: always  # 매번 강제 pull
```

또는:
```yaml
services:
  app:
    image: myregistry/app:${IMAGE_TAG}  # CD 에서 SHA 주입
```

## 의심될 때 — EC2 에서 확인 명령

CD 가 성공이라고 표시되는데 운영이 이상하면:

```bash
# 1. 컨테이너 살아있는지 + 언제 떴는지
docker ps
docker inspect my-app --format '{{.Created}}'

# 2. 부팅 로그
docker logs --tail 200 my-app

# 3. 이미지 SHA 확인
docker inspect my-app --format '{{.Image}}'
```

**핵심 휴리스틱**: `.Created` 시각 vs 마지막 CD 시각 비교.
- CD 가 5분 전 끝났는데 컨테이너가 1일 전 거면 → CD 가 silent fail
- 5분 전 시각이면 → 새 이미지로 떠 있음 ✅

## 발표 시 한 줄

> "잘 설정된 CD 는 명령 발사부터 컨테이너 health check 까지 자동입니다. 다만 silent fail 함정이 여러 층에 있어서, wait command-executed 와 health check 두 단계의 가시화가 안전망의 핵심입니다."

## FAQ

**Q: SSH 로 직접 docker pull 하는 게 더 빠른 거 아닌가?**
→ 한 번은 빠르지만 매번 사람이 하면 부담. 자동화하면 PR 머지 = 운영 반영. 사람의 실수 / 잊음 / 시간 차이 다 제거.

**Q: ECS 나 EKS 면 SSM 이 필요 없지 않나?**
→ 그쪽은 컨테이너 오케스트레이션이 service update 만 호출하면 자동 롤링 배포. SSM 은 단순 EC2 + Docker 조합용. ECS 가 더 클라우드 native 한 답.

**Q: 옛 CD 의 silent fail 진실은 어떻게 발견했나?**
→ wait command-executed 와 health check 를 처음 추가한 후 다음 release 가 1.5초만에 fail 했음. 이전엔 같은 함정이 있어도 CD 초록불이라 발견 못 했던 것. **가시화의 가치** — 안전망 추가 = 숨어 있던 진실 드러남.

## 관련

- [CI vs CD 브랜치 전략](ci-vs-cd-branch-strategy.md) — CD trigger 가 언제 일어나는지
