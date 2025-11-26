# 내가 경험한 Redis

---

## Redis 탄생 배경

```mermaid
timeline
    title Redis 탄생과 발전
    2009 : Salvatore Sanfilippo(antirez)
         : 이탈리아 시칠리아
         : LLOOGG 실시간 로그 분석 서비스 개발 중
         : MySQL 성능 한계 직면
    2010 : Redis 1.0 정식 출시
         : VMware 후원 시작
    2013 : Redis Sentinel (고가용성)
    2015 : Redis Cluster (수평 확장)
    2020 : antirez 프로젝트 리더 은퇴
         : Redis Labs 주도
    2024 : 라이선스 변경 (SSPL)
         : Valkey 포크 등장
```

### 왜 만들어졌나?

| 문제 상황 | Redis의 해결책 |
|----------|---------------|
| MySQL로 실시간 페이지뷰 집계 → 느림 | In-Memory 저장소로 μs 단위 응답 |
| 매번 디스크 I/O 발생 | 메모리에서 직접 처리 |
| 복잡한 쿼리 오버헤드 | 단순 Key-Value O(1) 연산 |
| 관계형 DB의 Lock 경쟁 | Single Thread로 Lock-free |

**핵심 철학**: "가장 빠른 데이터 접근은 디스크를 거치지 않는 것"

---

## Redis 기본 상식

```mermaid
flowchart TB
    subgraph Core["Redis 핵심 특성"]
        direction TB
        M[In-Memory] --> S[Single Thread]
        S --> D[다양한 자료구조]
        D --> P[Persistence 옵션]
    end

    subgraph Performance["성능 지표"]
        P1["읽기: 100,000+ ops/sec"]
        P2["쓰기: 80,000+ ops/sec"]
        P3["지연: < 1ms (99th percentile)"]
    end

    Core --> Performance
```

### 이름의 의미

**RE**mote **DI**ctionary **S**erver = 원격 딕셔너리 서버

### 핵심 특성 5가지

| 특성 | 설명 | 이점 |
|------|------|------|
| **In-Memory** | 모든 데이터를 RAM에 저장 | μs 단위 응답, 디스크 I/O 제거 |
| **Single Thread** | 명령을 순차 처리 | Lock 없음, 원자성 보장, 경쟁 상태 제거 |
| **다양한 자료구조** | String, Hash, List, Set, ZSet 등 | 용도별 최적 구조 선택 가능 |
| **Persistence** | RDB(스냅샷) / AOF(로그) | 재시작 시 데이터 복구 |
| **Replication** | Master-Replica 구조 | 읽기 분산, 장애 복구 |

### 자료구조별 시간복잡도

```mermaid
flowchart LR
    subgraph O1["O(1) 연산"]
        S1[STRING: GET/SET]
        H1[HASH: HGET/HSET]
        L1[LIST: LPUSH/RPOP]
        S2[SET: SADD/SISMEMBER]
    end

    subgraph ON["O(N) 연산"]
        L2[LIST: LRANGE]
        S3[SET: SMEMBERS]
        H2[HASH: HGETALL]
    end

    subgraph OLOG["O(log N) 연산"]
        Z1[ZSET: ZADD/ZRANK]
    end
```

### Single Thread인데 왜 빠른가?

```mermaid
flowchart TD
    subgraph Problem["멀티스레드 DB 문제"]
        T1[Thread 1] -->|Lock 대기| L[Lock]
        T2[Thread 2] -->|Lock 대기| L
        T3[Thread 3] -->|Lock 대기| L
        L --> D[(Data)]
    end

    subgraph Solution["Redis 해결책"]
        Q[명령 큐] --> E[Event Loop]
        E --> M[(Memory)]
        E --> R[결과 반환]
    end

    Problem -.->|"Lock 경쟁 = CPU 낭비"| X[느림]
    Solution -.->|"순차 처리 = 예측 가능"| Y[빠름]
```

| 요소 | 설명 |
|------|------|
| **I/O Multiplexing** | epoll/kqueue로 수천 연결 동시 처리 |
| **메모리 직접 접근** | 디스크 대기 없음 |
| **단순 연산** | 대부분 O(1), 복잡한 쿼리 파싱 없음 |
| **Context Switch 없음** | 스레드 전환 오버헤드 제거 |


## Redis 정의 (요약)

> Redis는 **In-Memory 기반의 Key-Value 자료구조 서버**로,
> O(1) 시간복잡도의 빠른 연산과 다양한 자료구조를 제공하여
> **캐싱, 세션 관리, 메시지 큐**에 가장 널리 사용된다.

---

## 본 문서의 구성

### 1. Cache

- [1계정의 무한 접속 차단](./2-cache.md#1-1계정의-무한-접속-차단)
- [i18n 다국어 캐싱](./2-cache.md#2-i18n)
- [메인 페이지 화면 구성요소](./2-cache.md#3-메인-페이지-화면-구성요소)

### 2. Queue

- [Job Queue 아키텍처](./3-queue.md)
