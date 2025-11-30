# Redis 기반 작업 큐 & 스케줄러 가이드

MSA 환경에서 Redis를 활용한 작업 큐(task queue) / 스케줄러(scheduler) 솔루션 정리

---

## 📊 전체 아키텍처 개요

```mermaid
flowchart LR
    subgraph Producers["🏭 Producers"]
        API[FastAPI Service]
        WEB[Web Service]
        CRON[Scheduler]
    end

    subgraph Redis["🔴 Redis"]
        LIST[(List/Queue)]
        STREAM[(Streams)]
        SORTED[(Sorted Set)]
    end

    subgraph Consumers["⚙️ Workers"]
        W1[Worker 1]
        W2[Worker 2]
        W3[Worker N]
    end

    API -->|LPUSH| LIST
    WEB -->|XADD| STREAM
    CRON -->|ZADD| SORTED

    LIST -->|BRPOP| W1
    STREAM -->|XREADGROUP| W2
    SORTED -->|ZPOPMIN| W3
```

---

## ✅ 대표 라이브러리 비교

```mermaid
flowchart TB
    subgraph Python["🐍 Python 생태계"]
        RQ["<b>RQ</b><br/>가장 단순, 경량"]
        RQS["<b>rq-scheduler</b><br/>RQ + cron 스케줄링"]
        HUEY["<b>Huey</b><br/>경량 + 재시도 + 결과저장"]
        DRAMA["<b>Dramatiq</b><br/>빠름, Redis/RabbitMQ"]
    end

    subgraph Other["🌐 기타 생태계"]
        SIDEKIQ["<b>Sidekiq</b><br/>Ruby/Rails 표준"]
    end

    RQ -.->|확장| RQS

    style RQ fill:#e1f5fe
    style HUEY fill:#e8f5e9
    style DRAMA fill:#fff3e0
```

### 라이브러리별 특징

| 이름 | 복잡도 | 스케줄링 | 재시도 | 결과저장 | 적합한 용도 |
|------|--------|----------|--------|----------|-------------|
| **RQ** | ⭐ | ❌ | ❌ | ✅ | 단순 백그라운드 작업 |
| **rq-scheduler** | ⭐⭐ | ✅ | ❌ | ✅ | 예약/반복 작업 |
| **Huey** | ⭐⭐ | ✅ | ✅ | ✅ | 소규모 서비스 올인원 |
| **Dramatiq** | ⭐⭐⭐ | ✅ | ✅ | ✅ | 성능 중시 서비스 |

---

## ⏰ rq-scheduler 동작 메커니즘

### 핵심 구조

```mermaid
flowchart LR
    subgraph Storage["🔴 Redis 저장소"]
        ZSET[("Sorted Set<br/>(rq:scheduler:scheduled_jobs)<br/>score = timestamp")]
        QUEUE[("List<br/>(rq:queue:default)<br/>실행 대기 작업")]
    end

    subgraph Processes["⚙️ 프로세스"]
        SCHED["Scheduler Process<br/>(polling loop)"]
        WORKER["RQ Worker"]
    end

    APP[Application] -->|"ZADD<br/>score=실행시각"| ZSET
    SCHED -->|"1초마다 ZRANGEBYSCORE<br/>(now 이하 작업 조회)"| ZSET
    SCHED -->|"LPUSH<br/>(실행 시점 도래)"| QUEUE
    QUEUE -->|"BRPOP"| WORKER
```

### Scheduler 프로세스 (polling loop)

```mermaid
flowchart TD
    START((시작)) --> POLL["ZRANGEBYSCORE<br/>scheduled_jobs 0 {now}"]
    POLL --> CHECK{실행할<br/>작업 있음?}
    CHECK -->|Yes| MOVE["ZREM + LPUSH<br/>→ RQ Queue로 이동"]
    CHECK -->|No| SLEEP["sleep(1초)"]
    MOVE --> REPEAT{반복 작업?}
    REPEAT -->|Yes| RESCHEDULE["다음 실행 시각으로<br/>ZADD 재등록"]
    REPEAT -->|No| SLEEP
    RESCHEDULE --> SLEEP
    SLEEP --> POLL
```

### 사용 예시

```python
from rq_scheduler import Scheduler
from datetime import datetime, timedelta

scheduler = Scheduler(connection=redis_conn)

# 특정 시점 실행
scheduler.enqueue_at(
    datetime(2024, 12, 1, 9, 0),  # 실행 시각
    my_task, arg1, arg2
)
# → ZADD rq:scheduler:scheduled_jobs 1733043600 <job_id>

# 지연 실행
scheduler.enqueue_in(
    timedelta(minutes=30),  # 30분 후
    my_task
)

# 반복 실행 (cron)
scheduler.cron(
    "0 9 * * *",  # 매일 09:00
    func=daily_report
)
```

### 내부 동작 (의사 코드)

```python
while True:
    now = time.time()

    # score(실행시각) <= now 인 작업들 조회
    due_jobs = redis.zrangebyscore(
        'rq:scheduler:scheduled_jobs', 0, now
    )

    for job_id in due_jobs:
        redis.zrem('rq:scheduler:scheduled_jobs', job_id)
        redis.lpush('rq:queue:default', job_id)

        # cron 작업이면 다음 실행시각 재등록
        if job.is_cron:
            next_run = calculate_next_run(job.cron_expr)
            redis.zadd('rq:scheduler:scheduled_jobs',
                      {job_id: next_run.timestamp()})

    time.sleep(1)
```

### 핵심 포인트

| 구성 요소 | 역할 | Redis 자료구조 |
|-----------|------|----------------|
| **Scheduler** | 1초마다 폴링, 실행시점 도래 작업을 큐로 이동 | - |
| **Scheduled Jobs** | 예약된 작업 저장 (score = timestamp) | `Sorted Set` |
| **RQ Queue** | 실행 대기 작업 | `List` |
| **Worker** | 큐에서 작업 꺼내 실행 | - |

> **Why Sorted Set?**
> - score를 timestamp로 활용 → 시간순 정렬 자동 보장
> - `ZRANGEBYSCORE`로 O(log N + M) 효율적 조회
> - Scheduler는 단일 프로세스로 실행 (중복 실행 방지 필요)

---

## 🔄 Redis 자체 구조 활용

### Redis List vs Streams 비교

```mermaid
flowchart TB
    subgraph List["📋 Redis List + BLPOP"]
        L1["LPUSH로 작업 추가"]
        L2["BRPOP으로 소비"]
        L3["단순 FIFO"]
        L1 --> L2 --> L3
    end

    subgraph Streams["📨 Redis Streams"]
        S1["XADD로 메시지 추가"]
        S2["Consumer Group 구성"]
        S3["XREADGROUP으로 소비"]
        S4["ACK/NACK 처리"]
        S5["재처리 메커니즘"]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    List -.->|"❌ 유실 위험"| RISK[프로세스 죽음 시<br/>작업 유실]
    Streams -.->|"✅ 내구성"| SAFE[메시지 보존<br/>재처리 가능]

    style RISK fill:#ffebee
    style SAFE fill:#e8f5e9
```

### 구현 방식 선택 기준

```mermaid
graph TD
    START{{"🎯 요구사항"}} --> Q1{메시지 유실<br/>허용 가능?}

    Q1 -->|Yes| Q2{구현 복잡도<br/>최소화?}
    Q1 -->|No| STREAMS["📨 Redis Streams<br/>+ Consumer Group"]

    Q2 -->|Yes| LIST["📋 Redis List<br/>+ BLPOP/BRPOP"]
    Q2 -->|No| LIB["📚 라이브러리 사용<br/>RQ, Huey 등"]

    style STREAMS fill:#c8e6c9
    style LIST fill:#fff9c4
    style LIB fill:#bbdefb
```

---

## 💡 설계 관점 체크리스트

```mermaid
mindmap
  root((MSA + Redis<br/>작업 큐 설계))
    언어/프레임워크
      다양한 서비스 → Redis Streams
      Python 단일 → RQ/Huey
    내구성 vs 속도
      속도 우선 → List + BLPOP
      내구성 우선 → Streams + ACK
    스케줄링
      cron 필요 → rq-scheduler
      지연 작업 → Sorted Set
    확장성
      소규모 → Redis만
      대규모 → Kafka/RabbitMQ 검토
```

---

## 🎯 FastAPI + Python 환경 의사결정 트리

```mermaid
flowchart TD
    START[["🚀 시작:<br/>FastAPI + Redis 환경"]]

    START --> Q1{현재 요구사항?}

    Q1 -->|단순 백그라운드 작업| A1["✅ RQ 추천<br/>최소 설정, 빠른 시작"]
    Q1 -->|예약/반복 작업 필요| A2["✅ RQ + rq-scheduler<br/>또는 Huey"]
    Q1 -->|메시지 유실 방지 필수| A3["✅ Redis Streams<br/>Consumer Group 구성"]
    Q1 -->|다국어 MSA 환경| A4["✅ Redis Streams + JSON<br/>언어 중립적"]

    A1 --> GROW{서비스 성장 시}
    A2 --> GROW
    A3 --> GROW
    A4 --> GROW

    GROW -->|작업량 폭증| MIGRATE["🔄 Kafka/RabbitMQ<br/>전환 검토"]
    GROW -->|안정적| KEEP["📌 현행 유지"]

    style START fill:#e3f2fd
    style A1 fill:#c8e6c9
    style A2 fill:#c8e6c9
    style A3 fill:#fff9c4
    style A4 fill:#fff9c4
    style MIGRATE fill:#ffecb3
```

---

## 📝 Quick Reference

### RQ 기본 사용

```python
from redis import Redis
from rq import Queue

redis_conn = Redis()
q = Queue(connection=redis_conn)

# 작업 등록
job = q.enqueue(my_function, arg1, arg2)
```

### Redis Streams Consumer Group

```python
import redis

r = redis.Redis()

# Consumer Group 생성
r.xgroup_create('mystream', 'mygroup', mkstream=True)

# 메시지 소비
messages = r.xreadgroup('mygroup', 'consumer1', {'mystream': '>'})

# ACK 처리
r.xack('mystream', 'mygroup', message_id)
```

---

## 🔗 참고 링크

- [RQ Documentation](https://python-rq.org/)
- [rq-scheduler](https://github.com/rq/rq-scheduler)
- [Huey](https://huey.readthedocs.io/)
- [Dramatiq](https://dramatiq.io/)
- [Redis Streams](https://redis.io/docs/data-types/streams/)
