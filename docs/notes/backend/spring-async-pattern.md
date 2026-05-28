# Spring 비동기 (`@EnableAsync` + `@Async`) — long-running 외부 호출 패턴

> **한 줄 요약**: `@EnableAsync` 는 스위치, `@Async` 는 표시 — 그 메서드는 별도 스레드 풀에서 실행되어 호출자를 블로킹하지 않는다. 외부 API 호출 / 무거운 계산처럼 5초 이상 걸리는 작업을 동기로 두면 사용자가 그대로 기다리니까 비동기로 떠넘기고, 결과는 큐 대신 **DB 상태 머신 (PENDING / DONE / FAILED)** 으로 전달 + 클라이언트가 폴링 — 작은 규모용 단순 구조.

## 문제 — 동기로 두면 생기는 4가지 문제

REST API 안에서 외부 서비스 호출이 5~30초 걸리는 경우 (AI 추론, 외부 결제 API, 이미지 변환 등):

```
POST /resource (사용자 요청)
  ├─ DB 저장 (50ms)
  ├─ 외부 호출 (5~30초)              ★ 병목
  └─ 201 응답
```

- **UX 최악** — 사용자가 5~30초 화면 멈춤
- **HTTP timeout 위험** — nginx / ALB / 클라이언트 측 30~60s 초과 가능성
- **서버 worker thread 고갈** — Tomcat thread 가 한 요청에 30초 묶임 → 동시 처리량 ↓
- **결합도 ↑** — 외부 서비스가 죽으면 본 API 자체가 실패

## 해법 — 비동기 + DB 상태 머신

```
POST /resource
  ├─ 본 리소스 DB 저장 (50ms)
  ├─ ChildResource(status=PENDING) row 같은 트랜잭션에 저장
  ├─ asyncService.trigger(resourceId)       ← @Async — 즉시 리턴
  └─ 201 응답 (총 60ms)

[별도 스레드 (백그라운드)]
  ├─ 외부 API 호출 (5~30초)
  └─ 결과를 PENDING row 에 UPDATE → DONE 또는 FAILED

[클라이언트]
  GET /resource/{id}/status   ← 폴링 (1~2초마다)
  └─ status 가 DONE 으로 바뀐 순간 결과 표시
```

→ 사용자는 즉시 201 받고 다음 화면 진행. 결과는 백그라운드로 천천히.

## `@EnableAsync` + `@Async` — Spring 이 실제로 하는 일

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

`@EnableAsync` 가 없으면 `@Async` 가 붙어 있어도 무시되고 그냥 동기 실행됨.

내부 메커니즘:
1. 시작 시 `@Async` 가 붙은 메서드를 가진 빈을 스캔
2. 그 빈을 **AOP 프록시 (proxy)** 로 감싼다
3. 프록시는 메서드 호출을 가로채서 **별도 스레드 풀에 실행을 위임**하고 즉시 리턴

```java
@Service
public class AsyncService {

    @Async
    public void trigger(UUID resourceId) {
        ChildResource resource = repository.findById(resourceId)...;
        try {
            ExternalResult result = client.call(...);
            resource.markDone(result);
        } catch (ExternalException e) {
            resource.markFailed(e.getMessage());
        }
    }
}
```

호출자 (컨트롤러) 가 `asyncService.trigger(id)` 를 부르면:
- 컨트롤러 스레드: 호출만 던지고 즉시 다음 줄로 진행 → 201 응답
- 별도 스레드: 위 메서드 본문을 실행

## 리턴 타입 — `void` vs `CompletableFuture<T>`

| 리턴 타입 | 의미 | 사용 시점 |
|---|---|---|
| `void` | Fire-and-forget — 호출자가 결과 / 예외 받을 방법 없음 | 결과를 DB 상태로 표현하는 경우 (이 패턴) |
| `CompletableFuture<T>` | 나중에 `.get()` 으로 결과 / 예외 받기 | 호출자가 결과를 직접 다뤄야 하는 경우 |

본 패턴은 `void` — 결과는 DB row 의 status 컬럼이 표현하니 호출자는 알 필요 없음.

## `ThreadPoolTaskExecutor` — 스레드 풀

`@Async` 호출마다 새 스레드를 만들면 비용 / 메모리 폭발. **풀에서 재사용**.

명시 안 하면 Spring 기본 executor (버전에 따라 동작 다름). 작은 규모는 기본값 충분, 본격 운영은 명시:

```java
@Bean(name = "externalExecutor")
public ThreadPoolTaskExecutor externalExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);        // 항상 살아있는 스레드 5개
    executor.setMaxPoolSize(20);        // 부하 시 최대 20개
    executor.setQueueCapacity(100);     // 대기 큐 100개
    executor.setThreadNamePrefix("external-");
    executor.initialize();
    return executor;
}
```

그 후 `@Async("externalExecutor")` 로 명시적 풀 지정 → 도메인별 풀 분리. 한 도메인 폭주가 다른 비동기 작업에 영향 X.

## 왜 큐 / 메시지 브로커 (Kafka / RabbitMQ / SQS) 안 쓰나

```java
public enum ResourceStatus {
    PENDING,  // 비동기 처리 중
    DONE,     // 성공
    FAILED    // 실패
}
```

→ **DB row 의 `status` 컬럼이 큐 역할**. 단순 상태 머신.

| 항목 | 큐 방식 | DB 상태 머신 |
|---|---|---|
| 인프라 | Kafka + Worker 별도 운영 | DB 만 |
| 재시도 | 큐 자체 기능 | 수동 구현 필요 |
| 확장성 | Worker 수평 확장 무한 | 같은 인스턴스 내 스레드 풀 |
| 적합 규모 | 대형 서비스 | **소규모 / 초기 단계** |
| 운영 복잡도 | 큐 모니터링 / DLQ / consumer lag | DB 한 곳만 |

작은 규모에서는 큐 인프라가 오버 엔지니어링. 트래픽 / 동시성이 늘면 그 때 도입 검토.

## ⚠️ 함정 — Race Condition: 트랜잭션 commit 전에 @Async 호출

**잘못된 예** (트랜잭션 안에서 호출):

```java
@Service
public class ResourceService {

    @Transactional
    public UUID create(...) {
        Resource resource = resourceRepository.save(...);
        childRepository.save(new ChildResource(PENDING));
        asyncService.trigger(resource.getId());  // ★ 트랜잭션 안에서 호출
        return resource.getId();
    }
}
```

이 코드는 다음 race 를 만든다:

1. 트랜잭션이 PENDING row 를 **메모리에는 만들었지만 DB commit 전**
2. `@Async trigger` 가 별도 트랜잭션에서 즉시 시작
3. 별도 트랜잭션이 PENDING row 를 SELECT — **아직 commit 안 됨 → 안 보임**
4. `NotFoundException` 발생

### 해결 — 컨트롤러에서 호출 (commit-then-trigger)

```java
@PostMapping("/resource")
public ResponseEntity<UUID> create(...) {
    UUID id = resourceService.create(...);
    // service.create() 의 @Transactional 가 메서드 return 시점에 commit.
    // 이 줄에서 호출하면 새 Async 트랜잭션이 PENDING row 를 안전하게 select 가능 (race 없음).
    asyncService.trigger(id);
    return ResponseEntity.status(HttpStatus.CREATED).body(id);
}
```

→ `service.create()` 가 return 할 때 트랜잭션 commit. 그 다음 줄에서 `trigger` 호출 → 비동기 스레드가 select 할 때 PENDING row 가 이미 DB 에 있음.

대안: `TransactionSynchronizationManager.registerSynchronization(...)` 로 `afterCommit` 훅에 등록 — 서비스 안에서도 안전하게 호출 가능. 다만 코드 복잡도 증가. 작은 규모는 컨트롤러 분리가 더 명확.

## ⚠️ 함정 — `@Async` 가 같은 클래스 내부 호출에서 안 먹음

```java
@Service
public class MyService {

    public void publicMethod() {
        this.asyncMethod();  // ❌ 동기 실행됨
    }

    @Async
    public void asyncMethod() { ... }
}
```

이유: Spring AOP 프록시는 **외부에서 빈 메서드를 호출할 때만** 가로채는 구조. 같은 클래스 안에서 `this.method()` 는 프록시를 거치지 않음.

해결: 비동기 메서드를 **다른 빈** 에 두고 호출. 위 예시처럼 컨트롤러 → 서비스, 또는 서비스 → 별도 비동기 서비스.

## 발표 시 한 줄

> "외부 호출이 5~30초 걸리면 동기로 둘 수 없으니 `@EnableAsync` + `@Async` 로 별도 스레드 풀에 떠넘기고, 결과는 큐 대신 DB 의 PENDING/DONE/FAILED 상태로 표현합니다. 트랜잭션 commit 전에 트리거하면 비동기 스레드가 PENDING row 를 못 찾는 race 가 생겨서, 컨트롤러에서 commit 직후 호출하는 게 안전합니다."

## FAQ

**Q: Kafka / RabbitMQ 안 쓰면 안 위험한가?**
→ 동시성 / 트래픽이 낮으면 DB 상태 머신이 충분히 안정적. 늘면 그 때 큐 도입. 처음부터 큐를 깔면 인프라 운영 부담 + 디버깅 복잡도가 비용으로 돌아옴.

**Q: 외부 호출이 실패하면 재시도는?**
→ 단순 패턴은 `markFailed` 만 호출하고 끝. 재시도는 수동 구현 (예: status=FAILED row 를 주기적으로 다시 trigger 하는 스케줄러). 자동 재시도가 본격 필요한 시점이 큐 도입의 자연스러운 트리거.

**Q: ThreadPool 이 가득 차면?**
→ Spring 기본은 사실상 무제한이라 메모리 폭발 위험. `ThreadPoolTaskExecutor` 명시 후 queueCapacity 까지 차면 `RejectedExecutionException` → 트리거 자체가 실패. 이 시점에는 "리소스는 저장됐는데 비동기 처리가 영원히 PENDING" 인 상황이 됨 — 부하 측정 후 정책 결정 (큐 도입 / pool 확장 / circuit breaker).

**Q: 클라이언트 폴링이 비효율 아닌가?**
→ 본격 운영에서는 WebSocket / SSE 로 push. 초기 단계는 폴링 (1~2초 간격) 으로 시작 + 후속에서 교체. 폴링도 `If-None-Match` / 304 응답 / Long-polling 으로 최적화 가능.

## 관련

- [AWS SSM Run Command — outbound polling](../ops/aws-ssm-outbound-polling.md) — 같은 "큐 + polling" 패턴이 인프라 영역에도 — 거기는 AWS SSM 이 큐, 여기는 DB
- [Hibernate `ddl-auto: update` 의 한계](../db/hibernate-ddl-auto-limits.md) — 비동기 1:1 정합성을 보장하는 UNIQUE 제약이 update 모드에서 누락 가능
