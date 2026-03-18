# 내가 경험한 OpenClaw — 7. 자동 에러 감지 & 수정

> **500 에러가 발생하면, AI가 알아서 찾아가서 고친다**

*6단락까지는 "사람이 시키면 에이전트가 한다"였다.
7단락은 한 단계 더 나아간다 — 에러가 발생하면 사람이 시키지 않아도, 에이전트가 스스로 감지하고 원인을 파악하고 수정까지 시도한다.
지금은 망가져서 동작하지 않고 있지만, 실제로 동작했던 파이프라인이다.*

---

## 7.1 전체 파이프라인

```mermaid
sequenceDiagram
    participant API as 🏢 Company A Dev<br/>API Servers
    participant CH as 📡 Service Hub<br/>API Server
    participant LOG as 💾 macmini-sub<br/>Log API Server
    participant OC as 🤖 macmini-sub<br/>OpenClaw
    participant FIX as 🏢 Company A Dev<br/>(SSH 원격 수정)

    Note over API,FIX: 자동 에러 감지 → 수정 파이프라인

    rect rgb(255, 235, 235)
        Note over API: 1️⃣ 에러 발생
        API->>API: HTTP 500 에러 발생
        API->>CH: HubErrorNotifier<br/>에러 로그 전송 (비차단)
    end

    rect rgb(235, 245, 255)
        Note over CH,LOG: 2️⃣ 에러 수집
        CH->>LOG: 에러 로그 저장<br/>(PostgreSQL)
        CH->>CH: Telegram 알림 발송
    end

    rect rgb(235, 255, 235)
        Note over OC,FIX: 3️⃣ 자동 분석 & 수정 (5분 주기)
        OC->>LOG: DB 조회: 새 에러 있나?
        LOG-->>OC: 새 에러 3건 발견

        OC->>FIX: SSH 원격 접속<br/>(company-a-dev-webserver-sky)
        OC->>FIX: 에러 로그 확인<br/>코드 분석
        OC->>FIX: 원인 파악 → 코드 수정
        OC->>FIX: docker restart → 검증
    end
```

---

## 7.2 에러 포워딩 — HubErrorNotifier

```mermaid
graph TB
    subgraph COMPANY_A["🏢 Company A Dev Servers"]
        PB["inference-agent-fastapi"]
        ST["media-storage-fastapi"]
        DN["project-d-qr-fastapi"]
        KM["project-d-kiosk-manager-fastapi"]
    end

    subgraph NOTIFIER["📡 HubErrorNotifier"]
        direction TB
        FILTER["HTTP 500만 필터링<br/>(4xx 제외)"]
        DEDUP["Debounce<br/>60초 내 동일 에러 제거"]
        PAYLOAD["페이로드 생성<br/>SHA1 핑거프린트<br/>module:function:line<br/>request_id + traceback"]
    end

    subgraph SERVICE_HUB["Service Hub API (port 8000)"]
        RECV["POST /api/v1/hub/telegram/send"]
        DB["PostgreSQL<br/>에러 로그 저장"]
        TG["Telegram 알림<br/>chat_id: <redacted>"]
    end

    PB -->|"500 에러"| NOTIFIER
    ST -->|"500 에러"| NOTIFIER
    DN -->|"500 에러"| NOTIFIER
    KM -->|"500 에러"| NOTIFIER

    FILTER --> DEDUP --> PAYLOAD
    PAYLOAD -->|"비차단 전송<br/>최대 3회 재시도"| RECV
    RECV --> DB
    RECV --> TG

    style COMPANY_A fill:#fff3e0,stroke:#ef6c00
    style NOTIFIER fill:#e3f2fd,stroke:#1565c0
    style SERVICE_HUB fill:#f3e5f5,stroke:#9c27b0
```

### 설계 원칙

| 원칙 | 구현 | 이유 |
|------|------|------|
| **비차단** | 포워딩 실패해도 API 응답에 영향 없음 | 모니터링이 서비스를 죽이면 안 됨 |
| **무한 재귀 방지** | 포워더 내부에서 로거 미사용 (stderr만) | 에러 전송 중 에러 → 무한 루프 방지 |
| **중복 제거** | 60초 내 동일 에러 debounce | Telegram 폭주 방지 |
| **추적성** | request_id + module:function:line + traceback | 정확한 위치 즉시 파악 |
| **재시도** | 최대 3회 retry | 네트워크 일시 장애 대응 |

---

## 7.3 자동 수정 파이프라인 — 5분 주기

```mermaid
graph TB
    subgraph CRON["⏰ macmini-sub OpenClaw (5분 주기)"]
        direction TB
        CHECK["DB 조회<br/>새 에러 메시지 확인"]
        FOUND{"새 에러<br/>있나?"}
        SSH["SSH 접속<br/>company-a-dev-webserver-sky"]
        ANALYZE["에러 로그 확인<br/>코드 분석"]
        FIX["원인 파악<br/>코드 수정"]
        VERIFY["docker restart<br/>검증"]
        REPORT["결과 기록<br/>Obsidian 노트"]
    end

    CHECK --> FOUND
    FOUND -->|"없음"| SKIP["스킵<br/>다음 5분 대기"]
    FOUND -->|"있음"| SSH
    SSH --> ANALYZE --> FIX --> VERIFY --> REPORT

    style CRON fill:#e8f5e9,stroke:#2e7d32
    style SKIP fill:#f5f5f5,stroke:#9e9e9e
```

### OpenClaw가 SSH로 수행하는 작업

```mermaid
sequenceDiagram
    participant OC as 🤖 OpenClaw<br/>(macmini-sub)
    participant DEV as 🏢 Company A Dev<br/>(SSH)

    OC->>DEV: ssh company-a-dev-webserver-sky

    rect rgb(230, 245, 255)
        Note over OC,DEV: 상태 확인
        OC->>DEV: git status -sb
        OC->>DEV: git log --since="6h" --oneline -n 30
        OC->>DEV: git diff --stat
        OC->>DEV: git diff -U2 | head -n 200
    end

    rect rgb(255, 245, 230)
        Note over OC,DEV: 대상 레포 순회
        OC->>DEV: project-d-qr-fastapi
        OC->>DEV: project-d-payments-fastapi
        OC->>DEV: project-d-kiosk-manager-fastapi
        OC->>DEV: hub-fastapi
        OC->>DEV: inference-agent-fastapi
    end

    rect rgb(230, 255, 230)
        Note over OC,DEV: 분석 결과 기록
        OC->>OC: Obsidian 노트에 append<br/>애매한 코드 / 결정필요 / 권장안
    end
```

---

## 7.4 데이터 흐름 — 3개 서버를 잇는 파이프라인

```mermaid
graph LR
    subgraph SERVER1["🏢 Company A Dev"]
        ERR["500 에러 발생"]
        CODE["소스 코드"]
    end

    subgraph SERVER2["📡 Service Hub"]
        RECV2["에러 수신"]
        DB2["PostgreSQL 저장"]
    end

    subgraph SERVER3["🏠 macmini-sub"]
        OC3["OpenClaw<br/>5분 주기 Cron"]
        FIX3["AI 에이전트<br/>분석 & 수정"]
    end

    ERR -->|"HTTP POST<br/>비차단"| RECV2
    RECV2 --> DB2
    DB2 -->|"DB 조회"| OC3
    OC3 -->|"새 에러 발견"| FIX3
    FIX3 -->|"SSH 원격 접속"| CODE

    style SERVER1 fill:#fff3e0,stroke:#ef6c00
    style SERVER2 fill:#f3e5f5,stroke:#9c27b0
    style SERVER3 fill:#e8f5e9,stroke:#2e7d32
```

### 이걸 안 했을 때

> 야간에 Inference Agent에서 500 에러가 터졌는데, 아침에 출근해서야 알았다.
> 로그를 뒤져서 원인 파악하고 수정하는 데 1시간.
> 이 파이프라인이 동작할 때는, 에러 발생 5분 안에 에이전트가 알아서 로그를 확인하고
> 코드를 분석해서 수정 방안까지 Obsidian에 기록해뒀다.
> 아침에 할 일은 기록을 확인하고 승인하는 것뿐이었다.

---

### 핵심 가치

| 전통적 방식 | 자동 에러 수정 파이프라인 |
|------------|------------------------|
| 에러 발생 → 아침에 발견 | 에러 발생 → 5분 내 감지 |
| 수동 로그 확인 → 원인 분석 | AI가 SSH로 접속 → 자동 분석 |
| 사람이 코드 수정 | AI가 수정안 제시 + 기록 |
| 반복되는 에러에 매번 대응 | debounce + 핑거프린트로 중복 제거 |
| 모니터링 ≠ 해결 | 모니터링 → 분석 → 수정을 하나로 |
