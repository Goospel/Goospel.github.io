# AWS SSM Run Command — outbound polling 메커니즘 + SSH 대체

> **한 줄 요약**: EC2 의 `amazon-ssm-agent` 가 outbound polling 으로 AWS 큐에서 명령을 받아 root 권한으로 실행. SSH 키 / 포트 22 인바운드 / private key 관리 모두 불필요 — IAM role 한 줄로 통제. 다만 `send-command` 는 enqueue 만 보장하므로 `wait command-executed` + health check 가 있어야 CD 가 진짜 deploy 성공을 의미.

## AWS SSM 이 무엇인가

**Systems Manager** — AWS 가 제공하는 **인스턴스 운영 통합 도구 모음**. 한 서비스가 아니라 여러 하위 기능의 묶음:

| 하위 기능 | 역할 |
|---|---|
| **Run Command** (`send-command`) | EC2 에 임의 shell 명령 원격 실행 |
| **Session Manager** | SSH 대체 — 브라우저 / CLI 로 EC2 shell 접속 (포트 22 안 열고) |
| **Parameter Store** | 시크릿 / 설정값 중앙 저장 |
| **Patch Manager** | OS 패치 자동화 |
| **State Manager** | EC2 의 desired state 유지 (config drift 방지) |
| **Inventory** | 모든 인스턴스의 설치 SW 목록 자동 수집 |

소규모 운영에서 가장 먼저 만나는 건 **Run Command** — GitHub Actions CD 에서 EC2 에 배포 명령 전달하는 용도.

## SSH 가 아니라 SSM 을 선택하는 이유

| 비교 항목 | SSH 방식 | SSM Run Command |
|---|---|---|
| **인증 키** | private key 파일 관리 필요 | IAM role 만 (키 X) |
| **포트** | 22 인바운드 오픈 필요 | **0 인바운드** (agent 가 outbound) |
| **감사 로그** | 직접 구성 (auditd 등) | CloudTrail 에 자동 |
| **GitHub Actions 연동** | SSH key 를 GitHub Secrets 에 박아야 함 | OIDC + IAM role 만 |
| **키 회전 / 유출 대응** | 키 새로 생성 + 모든 인스턴스 재배포 | IAM policy 한 줄 변경 |
| **포트 22 공격면** | 24/7 노출 | **0** |

CD 시나리오에서 결정적:
> **GitHub Actions 가 EC2 에 들어가야 하는데, SSH key 를 GitHub Secrets 에 저장하기 싫다.** 키가 새면 모든 EC2 의 authorized_keys 를 손봐야 함. SSM 은 IAM role 한 줄로 통제.

## SSM Run Command — 내부 작동 원리

핵심: **SSM Agent** 가 사전에 설치되어 있어야 한다 (Amazon Linux 2 / Ubuntu LTS 등 대부분의 AWS 공식 AMI 에는 기본 포함).

```
┌─────────────────────┐                  ┌──────────────────────────────┐
│ GitHub Actions      │                  │  EC2 instance                │
│ (CD workflow)       │                  │  ┌────────────────────────┐  │
│                     │                  │  │ amazon-ssm-agent       │  │
│ aws ssm             │                  │  │ (백그라운드 데몬)       │  │
│   send-command ──────► AWS SSM Service │  │                        │  │
│   --instance-ids    │  (Region 내부)    │  │ outbound polling       │  │
│   --document-name   │       │          │  │ (HTTPS, 443) ──────────┼──┘
│   AWS-RunShellScript│       └──────────► (큐에서 명령 받음)        │
│                     │                  │  │                        │  │
│ aws ssm             │                  │  │ shell 실행             │  │
│   wait              │                  │  │ (root 권한)            │  │
│   command-executed ◄─────── 결과 응답   │  │                        │  │
└─────────────────────┘                  │  └────────────────────────┘  │
                                         └──────────────────────────────┘
```

흐름 단계별:
1. **GitHub Actions** 가 `aws ssm send-command` 호출 — 인자: instance ID, document name (`AWS-RunShellScript`), 실행할 shell 명령
2. **AWS SSM 서비스** 가 명령을 큐에 박아둠
3. **EC2 의 ssm-agent** 가 **outbound polling** 으로 (인바운드 X) 큐를 주기적 확인
4. agent 가 명령을 가져와서 **root 권한**으로 shell 실행
5. 결과 (stdout / stderr / exit code) 를 SSM 서비스에 응답
6. GitHub Actions 의 `aws ssm wait command-executed` 가 그 응답을 받음

**핵심 포인트 — Outbound Polling**: EC2 입장에서는 **22번 포트가 닫혀 있어도 됨**. agent 가 HTTPS (443) 로 AWS endpoint 에 polling 만 함. 보안 그룹 인바운드를 완전히 봉쇄 가능.

## CD 워크플로우의 실제 모양

```yaml
- name: Deploy via SSM
  run: |
    COMMAND_ID=$(aws ssm send-command \
      --region ap-northeast-2 \
      --instance-ids "${{ secrets.EC2_INSTANCE_ID }}" \
      --document-name "AWS-RunShellScript" \
      --parameters 'commands=[
        "cd /home/ec2-user/my-app",
        "docker-compose pull",
        "docker-compose up -d"
      ]' \
      --query "Command.CommandId" \
      --output text)

    aws ssm wait command-executed \
      --command-id "$COMMAND_ID" \
      --instance-id "${{ secrets.EC2_INSTANCE_ID }}"

- name: Health check
  run: |
    for i in {1..30}; do
      if curl -fs http://${{ secrets.EC2_HOST }}:8080/health; then
        echo "Healthy!" && exit 0
      fi
      sleep 10
    done
    exit 1
```

## IAM Role 두 갈래 — 양쪽 다 필요

### GitHub Actions → AWS

GitHub Actions 가 OIDC 로 AWS 에 자격증명 받음. 그 role 의 policy:

```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:SendCommand",
    "ssm:GetCommandInvocation"
  ],
  "Resource": [
    "arn:aws:ec2:ap-northeast-2:*:instance/i-xxxxx",
    "arn:aws:ssm:ap-northeast-2::document/AWS-RunShellScript"
  ]
}
```

→ "이 한 EC2 인스턴스에만, 이 한 문서로만, send-command 가능". 최소 권한.

### EC2 → AWS SSM

EC2 의 **IAM instance profile** 에 `AmazonSSMManagedInstanceCore` policy 부착되어야 agent 가 polling 가능. 이게 없으면 ssm-agent 는 떠 있어도 명령을 못 받음 — 초기 셋업에서 가장 잘 빼먹는 한 줄.

## ⚠️ `send-command` 의 함정 — Silent Fail

`send-command` 는 **명령을 큐에 enqueue 한 순간 성공 반환**한다. 즉 agent 가 실제로 받아서 실행했는지는 **모름**:

```
옛 워크플로우 (안전망 없음):
  aws ssm send-command ...    ← enqueue 성공 → exit 0
                                  ✅ GitHub Actions 초록불

  실제 EC2 에서는:
    - agent 가 죽어 있으면 → 명령 영원히 안 받음
    - docker-compose plugin 없음 → 실행 실패
    - 둘 다 GitHub Actions 가 모름

  결과: CD 초록불 = 가짜. 운영 image 갱신은 사람이 수동 SSH 로
        (오랫동안 가짜 초록불 / 사람이 매 머지마다 손으로 docker pull)
```

### 안전망 — `wait command-executed` + health check

```bash
COMMAND_ID=$(aws ssm send-command ... --query "Command.CommandId" --output text)

aws ssm wait command-executed --command-id "$COMMAND_ID" --instance-id "..."
   # ↑ 명령이 Success / Failed 상태가 될 때까지 대기 → Failed 면 non-zero exit

curl -fsS http://EC2_HOST:8080/health --max-time 30
   # ↑ 헬스체크 endpoint 가 200 응답할 때까지 polling
```

→ **CD 초록불 = 진짜 deploy 성공** 으로 의미 회복.

`AWS SSM 자동화 범위` 노트 [CD 후 수동 작업 필요한가?](aws-ssm-automation-scope.md) 와 같이 읽으면 좋다 — 그쪽은 "자동화가 어디까지 되는가" 의 답, 이쪽은 "그 자동화가 어떤 메커니즘으로 작동하는가" 의 답.

## 추가 — `send-command` document 종류

`send-command` 는 통합 API. 실행 방식은 **SSM Document** 가 결정:

| Document | 용도 |
|---|---|
| `AWS-RunShellScript` | Linux shell 명령 |
| `AWS-RunPowerShellScript` | Windows PowerShell |
| `AWS-RunRemoteScript` | S3 의 스크립트 다운로드 후 실행 |
| `AWS-ApplyAnsiblePlaybooks` | Ansible playbook 실행 |

같은 API 로 OS 무관 명령 가능.

## 추가 — Session Manager (SSH 대체)

운영 중 디버깅 시 SSH 로 EC2 에 들어가는 대신:

```bash
aws ssm start-session --target i-xxxxx
```

- 브라우저 / AWS CLI 에서 직접 shell 접속
- 포트 22 완전 봉쇄 가능
- 모든 세션 CloudTrail 에 자동 로그
- IAM 으로 누가 언제 들어왔는지 통제

Run Command 로 자동화를 끝낸 다음의 자연스러운 보안 향상 단계.

## 발표 시 한 줄

> "SSH 키를 GitHub Secrets 에 두는 게 부담스러워서 SSM Run Command 를 선택했습니다. EC2 의 ssm-agent 가 outbound polling 으로 AWS 큐에서 명령을 받는 구조라 포트 22 를 닫고도 자동 배포가 됩니다. 다만 send-command 자체는 enqueue 만 보장해서 wait + health check 가 없으면 silent fail 이 일어납니다."

## FAQ

**Q: SSH 보다 느리지 않나요?**
→ Polling 주기 + 큐 처리로 명령 시작까지 보통 1~3초 지연. 자동 배포 시나리오에서는 무시 가능. 인터랙티브 디버깅에는 Session Manager 가 더 적합.

**Q: ssm-agent 가 죽으면?**
→ agent 가 죽으면 명령이 계속 큐에 쌓이다가 timeout. `wait command-executed` 가 fail 응답을 주니 CD 가 빨간불로 알림. 복구는 EC2 SSH (또는 콘솔의 EC2 Instance Connect) 로 들어가서 `sudo systemctl restart amazon-ssm-agent`.

**Q: ECS / EKS 면 SSM 이 필요 없지 않나?**
→ ECS / EKS 는 컨테이너 오케스트레이션이 service update 만 호출하면 자동 롤링 배포. SSM 은 **단순 EC2 + Docker** 조합용. 트래픽이 커지면 ECS / EKS 가 더 클라우드 native 한 답.

**Q: 명령 결과 stdout 은 어디서 보나요?**
→ `aws ssm get-command-invocation --command-id ... --instance-id ...` 로 stdout / stderr 조회. CloudWatch Logs 로 자동 스트리밍하는 옵션도 있음.

## 관련

- [AWS SSM 자동화 범위 — CD 후 수동 작업 필요한가?](aws-ssm-automation-scope.md) — 자동화 boundary 의 답 (이 노트는 그 메커니즘)
- [CI vs CD 브랜치 전략](ci-vs-cd-branch-strategy.md) — CD 트리거가 언제 일어나는지
- [Spring 비동기 패턴](../backend/spring-async-pattern.md) — 같은 "큐 + polling" 패턴이 도메인 비동기에도 (여기는 AWS SSM 이 큐, 거기는 DB 가 큐)
