# Redis Cache 활용 사례

---

# 1. 1계정의 무한 접속 차단

## 1. 미션

유료 계정으로 여럿이서 사용하는 편법을 막아주세요.

## 2. 설계 원칙

```mermaid
flowchart LR
    subgraph 요구사항["비즈니스 요구사항"]
        R1["동일 디바이스 타입 중복 로그인 차단"]
        R2["Desktop + Mobile 동시 허용"]
        R3["새 로그인 시 기존 세션 무효화"]
    end

    subgraph 기술결정["기술적 결정"]
        T1["Device Type 분리 키"]
        T2["JWT를 Value로 저장"]
        T3["TTL = JWT 만료시간"]
    end

    R1 --> T1
    R2 --> T1
    R3 --> T2
```

| 정책 | 설명 |
|------|------|
| **디바이스 분리** | Desktop과 Mobile을 별도로 관리하여 합리적 사용 허용 |
| **선점 방식** | 새 로그인이 기존 세션을 대체 (강제 로그아웃) |
| **실시간 검증** | 매 API 호출마다 Redis 토큰 비교 |

## 3. 아키텍처 다이어그램

```mermaid
flowchart TD
    subgraph Client["클라이언트"]
        Desktop["Desktop Browser"]
        Mobile["Mobile Browser"]
    end

    subgraph FastAPI["FastAPI SSO Server"]
        Login["로그인 엔드포인트<br/>/sign/login<br/>/sign_v2/login"]
        Verify["토큰 검증 엔드포인트<br/>/sign/me<br/>/sign_v2/me"]
        Logout["로그아웃 엔드포인트<br/>/sign/logout"]
    end

    subgraph Redis["Redis Session Store"]
        DKey["desktop+{user_id}:{JWT}"]
        MKey["mobile+{user_id}:{JWT}"]
    end

    Desktop -->|"1. 로그인 요청"| Login
    Mobile -->|"1. 로그인 요청"| Login

    Login -->|"2. User-Agent 분석"| UA["get_device_type()"]
    UA -->|"3. 디바이스별 세션 저장"| Redis

    Desktop -->|"4. API 요청 + Token"| Verify
    Mobile -->|"4. API 요청 + Token"| Verify

    Verify -->|"5. 저장된 토큰 조회"| Redis
    Verify -->|"6. 토큰 비교 검증"| Result{"토큰 일치?"}

    Result -->|"Yes"| Success["인증 성공"]
    Result -->|"No"| Fail["409 다른 기기에서 로그인"]

    Desktop -->|"7. 로그아웃"| Logout
    Logout -->|"8. 세션 삭제"| Redis
```

## 4. 세션 키 설계

```mermaid
flowchart LR
    subgraph KeyStructure["키 구조"]
        K["session:{device_type}:{user_id}"]
    end

    subgraph Examples["예시"]
        E1["session:desktop:12345"]
        E2["session:mobile:12345"]
    end

    subgraph Value["Value = JWT Token"]
        V["eyJhbGciOiJIUzI1NiIs..."]
    end

    KeyStructure --> Examples
    Examples --> Value
```

### Redis 명령어 흐름

```redis
# 로그인 시: 세션 저장 (기존 세션 자동 덮어쓰기)
SETEX session:desktop:12345 86400 "eyJhbGciOiJIUzI1NiIs..."

# API 호출 시: 토큰 검증
GET session:desktop:12345
# → 저장된 토큰과 요청 토큰 비교

# 로그아웃 시: 세션 삭제
DEL session:desktop:12345
```

## 5. TTL 전략

```mermaid
flowchart TD
    subgraph JWT["JWT 토큰"]
        J1["생성 시간: now()"]
        J2["만료 시간: now() + 24h"]
    end

    subgraph Redis["Redis TTL"]
        R1["SETEX key 86400 token"]
        R2["TTL = JWT 만료시간과 동기화"]
    end

    subgraph Lifecycle["세션 생명주기"]
        L1["로그인"] --> L2["활성"]
        L2 --> L3{"만료 조건"}
        L3 -->|"TTL 도달"| L4["자동 삭제"]
        L3 -->|"새 로그인"| L5["덮어쓰기"]
        L3 -->|"명시적 로그아웃"| L6["즉시 삭제"]
    end

    JWT --> Redis
    Redis --> Lifecycle
```

| 상황 | TTL 처리 |
|------|----------|
| **최초 로그인** | `SETEX` - JWT 만료시간과 동일하게 설정 |
| **재로그인 (동일 디바이스)** | 기존 키 덮어쓰기, TTL 갱신 |
| **다른 기기 로그인** | 기존 세션 유지, 새 키 생성 (device_type 다름) |
| **토큰 갱신 (Refresh)** | 새 토큰으로 Value 교체, TTL 리셋 |

## 6. 중복 로그인 감지 시퀀스

```mermaid
sequenceDiagram
    participant User1 as User (PC1)
    participant User2 as User (PC2)
    participant API as API Server
    participant Redis

    Note over User1,Redis: PC1에서 로그인
    User1->>API: POST /login
    API->>Redis: SETEX session:desktop:123 86400 "token_A"
    API-->>User1: 200 OK (token_A)

    Note over User1,Redis: PC2에서 동일 계정 로그인
    User2->>API: POST /login
    API->>Redis: SETEX session:desktop:123 86400 "token_B"
    Note over Redis: token_A → token_B 덮어쓰기
    API-->>User2: 200 OK (token_B)

    Note over User1,Redis: PC1에서 API 호출 시도
    User1->>API: GET /api/data (token_A)
    API->>Redis: GET session:desktop:123
    Redis-->>API: "token_B"
    API->>API: token_A ≠ token_B
    API-->>User1: 409 Conflict "다른 기기에서 로그인됨"
```

---

# 2. i18n 다국어 캐싱

## 1. 접근 방식 비교

```mermaid
flowchart TB
    subgraph Approach1["방식 1: Next.js JSON 파일"]
        A1_1["/locales/ko.json"]
        A1_2["/locales/en.json"]
        A1_3["/locales/ja.json"]
        A1_4["빌드 시 번들링"]
    end

    subgraph Approach2["방식 2: API + Redis 캐싱"]
        A2_1["DB: i18n_translations"]
        A2_2["Admin: 번역 관리"]
        A2_3["Redis: 캐시 레이어"]
        A2_4["API: /api/i18n/:lang"]
    end

    A1_1 --> A1_4
    A1_2 --> A1_4
    A1_3 --> A1_4

    A2_1 --> A2_2
    A2_2 --> A2_3
    A2_3 --> A2_4
```

### 방식별 장단점

| 항목 | JSON 파일 방식 | API + Redis 방식 |
|------|---------------|-----------------|
| **수정 반영** | 재빌드/재배포 필요 | 즉시 반영 가능 |
| **개발자 개입** | 매번 필요 | 최소화 |
| **변경 이력** | Git 히스토리 | DB 테이블 추적 |
| **초기 로딩** | 빠름 (번들 포함) | API 호출 필요 |
| **적합한 상황** | 정적 서비스 | 빈번한 문구 변경 |

## 2. API + Redis 아키텍처

```mermaid
flowchart TD
    subgraph Admin["관리자 영역"]
        AdminUI["Admin Dashboard"]
        AdminAPI["Admin API"]
    end

    subgraph Backend["Backend"]
        DB[(PostgreSQL<br/>i18n_translations)]
        Sync["동기화 스케줄러<br/>(주기적 or 이벤트)"]
        Redis[(Redis Cache)]
        API["i18n API Server"]
    end

    subgraph Frontend["Frontend"]
        Next["Next.js App"]
        Hook["useTranslation Hook"]
    end

    AdminUI -->|"번역 수정"| AdminAPI
    AdminAPI -->|"UPDATE/INSERT"| DB
    AdminAPI -->|"캐시 무효화 이벤트"| Sync

    Sync -->|"전체 동기화"| DB
    Sync -->|"HSET i18n:{lang}"| Redis

    Next -->|"GET /api/i18n/ko"| API
    API -->|"HGETALL i18n:ko"| Redis
    API -->|"Cache Miss 시"| DB
    API -->|"응답"| Next
    Next --> Hook
```

## 3. Redis 데이터 구조

```mermaid
flowchart LR
    subgraph Keys["Redis Hash Keys"]
        K1["i18n:ko"]
        K2["i18n:en"]
        K3["i18n:ja"]
    end

    subgraph Fields["Hash Fields (Key: Value)"]
        F1["common.welcome: 환영합니다"]
        F2["common.logout: 로그아웃"]
        F3["error.notFound: 찾을 수 없습니다"]
        F4["button.submit: 제출"]
    end

    K1 --> F1
    K1 --> F2
    K1 --> F3
    K1 --> F4
```

### Redis 명령어

```redis
# 언어별 번역 데이터 저장 (Hash)
HSET i18n:ko common.welcome "환영합니다" common.logout "로그아웃"
HSET i18n:en common.welcome "Welcome" common.logout "Logout"

# 특정 키 조회
HGET i18n:ko common.welcome

# 전체 언어 데이터 조회
HGETALL i18n:ko

# 캐시 갱신 시 TTL 설정 (선택적)
EXPIRE i18n:ko 3600
```

## 4. 동기화 전략

```mermaid
sequenceDiagram
    participant Admin as Admin UI
    participant API as Admin API
    participant DB as PostgreSQL
    participant Scheduler as Sync Scheduler
    participant Redis

    Note over Admin,Redis: 방식 1: 즉시 동기화
    Admin->>API: PUT /admin/i18n/ko/common.welcome
    API->>DB: UPDATE translations SET value='...'
    API->>Redis: HSET i18n:ko common.welcome "..."
    API-->>Admin: 200 OK

    Note over Admin,Redis: 방식 2: 배치 동기화
    Scheduler->>Scheduler: 매 5분 실행
    Scheduler->>DB: SELECT * FROM translations WHERE updated_at > last_sync
    DB-->>Scheduler: 변경된 번역 목록
    Scheduler->>Redis: HSET i18n:{lang} {key} {value} (다건)
    Scheduler->>Scheduler: last_sync = now()
```

| 동기화 방식 | 트리거 | 장점 | 단점 |
|------------|--------|------|------|
| **즉시 동기화** | DB 변경 시 | 실시간 반영 | 트래픽 증가 |
| **배치 동기화** | 주기적 (5분) | 효율적 | 지연 발생 |
| **이벤트 기반** | Admin 저장 버튼 | 명시적 | 수동 개입 |

---

# 3. 메인 페이지 화면 구성요소 캐싱

## 1. 캐싱 대상

```mermaid
mindmap
  root((메인 페이지<br/>캐싱 대상))
    Rankings
      실시간 인기 상품
      주간 판매 TOP 10
      카테고리별 순위
    Statistics
      총 판매량
      신규 가입자 수
      오늘의 거래액
    Charts
      매출 추이 그래프
      카테고리 점유율
      시간대별 트래픽
    Recommendations
      맞춤 추천 상품
      연관 상품
      최근 본 상품
```

## 2. 캐싱 아키텍처

```mermaid
flowchart TD
    subgraph Scheduler["스케줄러 (Batch Job)"]
        Cron["Cron: */5 * * * *"]
        Calculator["데이터 계산 로직"]
    end

    subgraph DataSources["데이터 소스"]
        DB[(PostgreSQL)]
        ES[(Elasticsearch)]
        Analytics["Analytics API"]
    end

    subgraph Redis["Redis Cache Layer"]
        R1["main:rankings:popular"]
        R2["main:rankings:weekly"]
        R3["main:stats:sales"]
        R4["main:charts:revenue"]
    end

    subgraph API["API Server"]
        Endpoint["/api/main/components"]
    end

    subgraph Frontend["Frontend"]
        MainPage["메인 페이지"]
    end

    Cron -->|"트리거"| Calculator
    Calculator -->|"SELECT/집계"| DB
    Calculator -->|"검색/분석"| ES
    Calculator -->|"메트릭 수집"| Analytics

    Calculator -->|"SET/HSET"| Redis

    MainPage -->|"GET"| Endpoint
    Endpoint -->|"GET/HGETALL"| Redis
    Endpoint -.->|"Cache Miss (fallback)"| DB
```

## 3. Redis 키 설계

```mermaid
flowchart LR
    subgraph Pattern["키 네이밍 패턴"]
        P1["main:{category}:{type}:{variant}"]
    end

    subgraph Examples["예시"]
        E1["main:rankings:popular:all"]
        E2["main:rankings:popular:category_1"]
        E3["main:stats:daily:2024-01-15"]
        E4["main:charts:revenue:weekly"]
    end

    Pattern --> Examples
```

### 데이터 타입별 Redis 구조

| 데이터 | Redis 타입 | 키 예시 | 이유 |
|--------|-----------|---------|------|
| **인기 순위** | Sorted Set | `main:rankings:popular` | 점수 기반 정렬 필요 |
| **통계 수치** | Hash | `main:stats:daily` | 다중 필드 관리 |
| **차트 데이터** | String (JSON) | `main:charts:revenue` | 복잡한 구조 직렬화 |
| **추천 목록** | List | `main:recommendations:{user_id}` | 순서 보장 |

### Redis 명령어

```redis
# 인기 순위 (Sorted Set) - 점수 = 판매량
ZADD main:rankings:popular 1500 "product:101" 1200 "product:102" 980 "product:103"

# 상위 10개 조회 (내림차순)
ZREVRANGE main:rankings:popular 0 9 WITHSCORES

# 일일 통계 (Hash)
HSET main:stats:daily total_sales 15000000 new_users 342 orders 1876

# 차트 데이터 (JSON String)
SET main:charts:revenue '{"labels":["Mon","Tue","Wed"],"data":[100,150,120]}'

# TTL 설정 (5분)
EXPIRE main:rankings:popular 300
```

## 4. 갱신 주기 전략

```mermaid
flowchart TD
    subgraph Frequency["갱신 빈도"]
        F1["실시간<br/>(1분 이내)"]
        F2["준실시간<br/>(5분)"]
        F3["배치<br/>(1시간)"]
        F4["일배치<br/>(1일)"]
    end

    subgraph Data["데이터 종류"]
        D1["실시간 접속자 수"]
        D2["인기 순위<br/>판매 통계"]
        D3["차트 데이터<br/>추천 목록"]
        D4["주간/월간 리포트"]
    end

    F1 --> D1
    F2 --> D2
    F3 --> D3
    F4 --> D4
```

| 갱신 주기 | 대상 데이터 | 구현 방식 | TTL |
|----------|------------|----------|-----|
| **1분** | 실시간 접속자, 현재 거래 | Pub/Sub + 증분 업데이트 | 60초 |
| **5분** | 인기 순위, 판매 통계 | Cron Job | 300초 |
| **1시간** | 차트 데이터, 추천 목록 | Batch Scheduler | 3600초 |
| **1일** | 주간/월간 집계 | Daily Batch | 86400초 |

## 5. Cache Miss 처리

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Redis
    participant DB

    Client->>API: GET /api/main/rankings
    API->>Redis: GET main:rankings:popular

    alt Cache Hit
        Redis-->>API: 캐시 데이터
        API-->>Client: 200 OK (from cache)
    else Cache Miss
        Redis-->>API: nil
        API->>DB: SELECT ... ORDER BY sales DESC LIMIT 10
        DB-->>API: 순위 데이터
        API->>Redis: SETEX main:rankings:popular 300 "{data}"
        API-->>Client: 200 OK (from DB)
    end
```

### Fallback 전략

```mermaid
flowchart TD
    Request["API 요청"] --> Check{"Redis 조회"}

    Check -->|"Hit"| Return["캐시 데이터 반환"]
    Check -->|"Miss"| DB["DB 조회"]

    DB --> Cache["Redis 저장"]
    Cache --> Return

    DB -->|"DB 장애"| Stale{"Stale 캐시 존재?"}
    Stale -->|"Yes"| StaleReturn["만료된 캐시 반환<br/>(Stale-While-Revalidate)"]
    Stale -->|"No"| Default["기본값 반환<br/>또는 503 에러"]
```

| 장애 상황 | 대응 전략 |
|----------|----------|
| **Redis 장애** | DB 직접 조회 (성능 저하 감수) |
| **DB 장애** | 만료된 캐시라도 반환 (Stale 캐시) |
| **양쪽 장애** | 정적 기본값 또는 503 응답 |

---

# 4. 캐싱 공통 고려사항

## Cache Invalidation 전략

```mermaid
flowchart LR
    subgraph Strategies["무효화 전략"]
        S1["TTL 기반<br/>시간 만료"]
        S2["Event 기반<br/>데이터 변경 시"]
        S3["Manual<br/>관리자 수동"]
    end

    subgraph Patterns["패턴"]
        P1["Write-Through"]
        P2["Write-Behind"]
        P3["Cache-Aside"]
    end

    S1 --> P3
    S2 --> P1
    S3 --> P2
```

## 모니터링 지표

| 지표 | 설명 | 정상 범위 |
|------|------|----------|
| **Hit Rate** | 캐시 적중률 | > 90% |
| **Memory Usage** | Redis 메모리 사용량 | < 80% |
| **Latency** | 응답 지연 시간 | < 1ms (p99) |
| **Eviction** | 메모리 부족으로 삭제된 키 | 0 (권장) |

```redis
# 캐시 적중률 확인
INFO stats
# keyspace_hits / (keyspace_hits + keyspace_misses) * 100

# 메모리 사용량 확인
INFO memory
# used_memory_human
```
