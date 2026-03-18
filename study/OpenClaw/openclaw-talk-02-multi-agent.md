# 내가 경험한 OpenClaw — 2. Multi Agent

> **OpenClaw 1대, Telegram Bot 6개 — 맥락이 끊기지 않는 멀티 에이전트 워크플로우**

---

## 2.1 왜 멀티 에이전트인가?

### 이걸 안 했을 때

> 처음에는 봇 하나로 모든 걸 시켰다.
> CompanyA A-1 프로젝트 코드리뷰를 하다가 CompanyB 배포 이슈를 물어보면, 에이전트가 A-1 프로젝트 파일에서 CompanyB 관련 코드를 찾으려고 했다.
> 대화가 길어질수록 이전 맥락이 희석되고, 여러 프로젝트의 컨텍스트가 뒤섞여서 결국 "처음부터 다시 설명"하는 일이 반복됐다.

AI 에이전트에게 모든 업무를 한 채팅에서 시키면 어떻게 될까?

```mermaid
graph LR
    subgraph SINGLE["❌ 단일 에이전트"]
        U1["A-1 서버 로그 확인해줘"]
        U2["A-b 빌드 배포해줘"]
        U3["내일 일정 알려줘"]
        U4["CompanyB DB 덤프해줘"]
        BOT["🤖 하나의 Bot"]
    end

    U1 --> BOT
    U2 --> BOT
    U3 --> BOT
    U4 --> BOT

    BOT -->|"컨텍스트 혼재<br/>맥락 유실<br/>토큰 낭비"| FAIL["😵 품질 저하"]

    style SINGLE fill:#fff0f0,stroke:#e53935
    style FAIL fill:#ffcdd2,stroke:#e53935
```

```mermaid
graph LR
    subgraph MULTI["✅ 멀티 에이전트"]
        U1["A-1 서버 로그 확인해줘"] --> B1["🤖 w-company-a4sky"]
        U2["A-b 빌드 배포해줘"] --> B2["🤖 w-company-b4sky"]
        U3["내일 일정 알려줘"] --> B3["🤖 planner4sky"]
        U4["CompanyB DB 덤프해줘"] --> B4["🤖 w-lgt4sky"]
    end

    B1 -->|"A-1 맥락 유지"| OK1["✅"]
    B2 -->|"A-b 맥락 유지"| OK2["✅"]
    B3 -->|"일정 맥락 유지"| OK3["✅"]
    B4 -->|"CompanyB 맥락 유지"| OK4["✅"]

    style MULTI fill:#f0fff0,stroke:#43a047
```

**요점:** 같은 CompanyA 소속이라도 프로젝트별로 에이전트를 나누면 맥락이 섞이지 않고, 대화가 길어져도 정확도가 유지된다.

---

## 2.2 에이전트 구성 — 1 Gateway, 6 Agents

```mermaid
graph TB
    subgraph GW["🖥️ OpenClaw Gateway (1대)"]
        direction TB
        ENGINE["Agent Engine<br/>openai-codex/gpt-5.2"]
        ROUTER["Binding Router<br/>accountId → agentId"]
    end

    subgraph WORK["🔧 업무 에이전트 (w-)"]
        WP["w-company-a<br/>@company-a4sky_bot<br/>CompanyA / A-1 프로젝트"]
        WC["w-company-b<br/>@company-b4sky_bot<br/>CompanyA / A-b 프로젝트"]
        WL["w-lgt<br/>@lgt4sky_bot<br/>CompanyB 프로젝트"]
    end

    subgraph LIFE["🧑 개인 에이전트"]
        PL["planner<br/>@planner4sky_bot<br/>일정 관리 (Luma)"]
    end

    subgraph DEFAULT["📌 기본"]
        MN["main<br/>@sky4clawd_bot<br/>범용"]
    end

    ENGINE --> ROUTER
    ROUTER --> WP
    ROUTER --> WC
    ROUTER --> WL
    ROUTER --> PL
    ROUTER --> MN

    style GW fill:#f3e5f5,stroke:#9c27b0
    style WORK fill:#e3f2fd,stroke:#1976d2
    style LIFE fill:#fff8e1,stroke:#f9a825
    style DEFAULT fill:#f5f5f5,stroke:#757575
```

### 에이전트별 역할과 성격

| 에이전트 | Bot | 도메인 | 성격 | 주요 업무 |
|---------|-----|--------|------|-----------|
| **w-company-a** | @company-a4sky_bot | CompanyA / A-1 | 업무형 | Service Hub, TTS, Kiosk, 서버 운영 |
| **w-company-b** | @company-b4sky_bot | CompanyA / A-b | 업무형 (paul) | CMS/Payload, Products 관리, Algolia 검색, AWS EC2 |
| **w-lgt** | @lgt4sky_bot | CompanyB | 간결/주도적 (paul) | API, DB 덤프/복구, WSL/Docker, 클라이언트 그래프 |
| **planner** | @planner4sky_bot | 일정/계획 | 차분/다정 (Luma) | TODO, 리마인더, 프로젝트 관리, 일정 조율 |
| **main** | @sky4clawd_bot | 범용 | 기본 | DM 전용, 특정 도메인에 속하지 않는 작업 |

---

## 2.3 컨텍스트 격리 아키텍처

각 에이전트는 **완전히 독립된 환경**에서 동작한다.

| 계층 | w-company-a | w-company-b | w-lgt | planner | main |
|------|-----------|---------|-------|---------|------|
| **Workspace** | `workspace-w-company-a/` | `workspace-w-company-b/` | `workspace-w-lgt/` | `workspace-planner/` | `clawd/` |
| **Sessions** | `agents/w-company-a/sessions/` | `agents/w-company-b/sessions/` | `agents/w-lgt/sessions/` | `agents/planner/sessions/` | `agents/main/sessions/` |
| **Memory** | `w-company-a.sqlite` | `w-company-b.sqlite` | `w-lgt.sqlite` | `planner.sqlite` | `main.sqlite` |
| **Ontology** | `graph.sqlite` | `graph.sqlite` | `graph.sqlite` | `graph.sqlite` | — |
| **Config** | AGENTS.md / SOUL.md | AGENTS.md / SOUL.md | AGENTS.md / SOUL.md | AGENTS.md / SOUL.md | CLAUDE.md |

> 모든 행이 에이전트별로 **물리적으로 분리**되어 있다. 데이터가 섞일 여지가 없다.

### 격리 계층

```mermaid
graph TB
    subgraph LAYERS["컨텍스트 격리 4계층"]
        L1["1️⃣ Telegram Bot<br/>봇 선택 = 도메인 선택"]
        L2["2️⃣ Workspace<br/>파일/코드 물리적 분리"]
        L3["3️⃣ Session<br/>대화 기록 독립 저장"]
        L4["4️⃣ Memory<br/>SQLite + 일일 로그 분리"]
    end

    L1 --> L2 --> L3 --> L4

    style LAYERS fill:#fafafa,stroke:#616161
```

| 계층 | 무엇이 분리되나 | 왜 필요한가 |
|------|----------------|------------|
| **Telegram Bot** | 입력 채널 자체 | 봇을 고르는 순간 도메인이 결정됨 |
| **Workspace** | 파일, 코드, 설정 | 프로젝트 코드가 섞이지 않음 |
| **Session** | 대화 이력 | 과거 맥락이 다른 주제에 오염되지 않음 |
| **Memory** | 장기 기억 (벡터 DB + 일일 로그) | 에이전트가 해당 도메인만 기억함 |


---

## 2.4 Binding Router — 메시지가 에이전트를 찾아가는 방법

```mermaid
sequenceDiagram
    participant User as 📱 사용자
    participant TG as Telegram
    participant GW as OpenClaw Gateway
    participant Router as Binding Router
    participant Agent as 해당 에이전트

    User->>TG: @company-a4sky_bot에 메시지 전송 (A-1 프로젝트)
    TG->>GW: webhook / polling
    GW->>Router: accountId: "w-company-a" 확인

    rect rgb(230, 245, 255)
        Note over Router: bindings 규칙 매칭
        Router->>Router: channel: telegram<br/>accountId: w-company-a<br/>→ agentId: w-company-a
    end

    Router->>Agent: w-company-a 에이전트에 라우팅
    Agent->>Agent: workspace-w-company-a 컨텍스트 로드
    Agent->>Agent: w-company-a 세션 & 메모리 참조
    Agent-->>User: A-1 프로젝트 맥락의 응답
```

### openclaw.json 바인딩 설정

```json
"bindings": [
  { "agentId": "w-company-a", "match": { "channel": "telegram", "accountId": "w-company-a" } },
  { "agentId": "w-company-b",   "match": { "channel": "telegram", "accountId": "w-company-b" } },
  { "agentId": "w-lgt",     "match": { "channel": "telegram", "accountId": "w-lgt" } },
  { "agentId": "planner",   "match": { "channel": "telegram", "accountId": "planner" } }
]
```

규칙은 단순하다. 그런데 이 정도만으로도 **하나의 게이트웨이가 5개의 독립된 AI 에이전트처럼 동작**한다.

---

## 2.5 메모리 시스템 — 맥락이 유지되는 비결

각 에이전트는 독립된 메모리를 가진다. 핵심은 **워크스페이스별 격리**.

| 계층 | 저장소 | 역할 |
|------|--------|------|
| Daily Log | `memory/YYYY-MM-DD.md` | 당일 작업 raw 기록 |
| MEMORY.md | 큐레이팅된 장기 기억 | 중요 결정/패턴만 보존 |
| Vector DB | `memory/{agent}.sqlite` | 임베딩 기반 시맨틱 검색 |
| Ontology | `memory/ontology/graph.sqlite` | 엔티티/관계 구조화 |

> **w-company-a**(A-1)에게 "Inference Agent 성별 검출 로직 확인해줘"라고 말하면 과거 기록을 찾아 응답하지만,
> **w-company-b**(A-b)에게 같은 질문을 하면 **아무 기억이 없다.** 같은 CompanyA 소속이라도 프로젝트별 격리 덕분이다.
>
> *메모리 시스템의 상세 구조는 [4단락 Obsidian & Ontology](openclaw-talk-04-obsidian-ontology.md)에서 다룬다.*

---

## 2.6 실제 운영 패턴 — 하루 동안의 봇 전환

```mermaid
sequenceDiagram
    participant User as 📱 사용자
    participant WP as 🔧 w-company-a (A-1)
    participant WC as 🔧 w-company-b (A-b)
    participant WL as 🔧 w-lgt (CompanyB)
    participant PL as 📅 planner

    Note over User: 오전 9시 — 업무 시작
    User->>PL: 오늘 할 일 알려줘
    PL-->>User: 1. A-1 TTS 캐시 수정<br/>2. A-b Products 핫픽스<br/>3. CompanyB DB 점검

    Note over User: 오전 10시 — CompanyA A-1 작업
    User->>WP: TTS 캐시 해시 비교 로직 확인해줘
    WP-->>User: ttsPublicCache.ts 분석 결과...
    User->>WP: 서버에서 Inference Agent 로그 확인해줘
    WP-->>User: SSH 접속 → 로그 내용...

    Note over User: 오후 2시 — CompanyA A-b 작업
    User->>WC: Products publish 상태 확인해줘
    WC-->>User: Algolia 동기화 이슈 발견...

    Note over User: 오후 4시 — CompanyB 작업
    User->>WL: DB 덤프 떠줘
    WL-->>User: WSL SSH 접속 → pg_dump 완료...

    Note over User,PL: 각 봇은 자기 도메인의<br/>맥락만 기억하고 유지
```

---

## 2.7 핵심 가치

> **"맥락 전환 비용을 제로로 만든다."**
>
> CompanyA의 두 프로젝트(A-1, A-b) + CompanyB + 일정 관리를 하나의 AI로 돌리면 맥락이 뒤섞인다.
> Telegram Bot을 프로젝트별로 분리하면, **봇을 선택하는 것 자체가 컨텍스트 스위칭**이 된다.
> 같은 회사 소속이라도 프로젝트가 다르면 에이전트를 나눈다 — 각자 자기 도메인만 기억하고, 서로 오염되지 않는다.

| 문제 | 멀티 에이전트 해결 |
|------|-------------------|
| 대화가 길어지면 맥락 유실 | 도메인별 세션 분리로 맥락 보존 |
| 프로젝트 간 컨텍스트 혼재 | Workspace + Memory 물리적 격리 |
| "아까 그거" 참조 실패 | Vector DB로 과거 대화 시맨틱 검색 |
| 업무/개인 경계 모호 | w- 접두사 = 업무, 나머지 = 개인 |
| 에이전트 성격 획일화 | 봇별 Identity 설정 (paul, Luma) |

---

*다음 단락: 3. OMX Runbook*
