# 내가 경험한 OpenClaw — 4. Obsidian & Ontology

> **기계가 읽는 그래프 + 사람이 읽는 노트 — 하이브리드 메모리 아키텍처**

*Runbook으로 분석하고, 에이전트가 작업한 결과는 어디에 쌓이는가?
AI가 쿼리할 수 있는 구조화된 그래프(Ontology)와, 사람이 읽을 수 있는 마크다운 노트(Obsidian PARA). 이 두 가지를 같이 굴린다.*

---

## 4.1 왜 두 가지 저장소가 필요한가?

```mermaid
graph TB
    subgraph DUAL["하이브리드 메모리"]
        direction LR

        subgraph ONTOLOGY["🔗 Ontology (SQLite)"]
            O1["구조화된 지식 그래프"]
            O2["타입 + 관계 + 제약"]
            O3["AI가 쿼리하기 최적"]
        end

        subgraph PARA["📝 PARA (Obsidian)"]
            P1["자유형식 마크다운"]
            P2["프로젝트 / 영역 / 자료 / 보관"]
            P3["사람이 읽고 쓰기 최적"]
        end
    end

    INPUT["데이터 수집<br/>Telegram / Claude Code"]
    INPUT -->|"구조화"| ONTOLOGY
    INPUT -->|"인간화"| PARA

    ONTOLOGY -->|"AI가 맥락 검색"| AGENT["🤖 에이전트 의사결정"]
    PARA -->|"사람이 리뷰"| HUMAN["👤 사용자 확인"]

    style ONTOLOGY fill:#e3f2fd,stroke:#1565c0
    style PARA fill:#fff8e1,stroke:#f9a825
```

| | Ontology | PARA (Obsidian) |
|---|---------|----------------|
| **형식** | SQLite 그래프 DB | 마크다운 파일 |
| **대상** | AI 에이전트 | 사람 |
| **구조** | Entity → Relation → Entity | 폴더 → 파일 → 본문 |
| **강점** | 정확한 쿼리, 관계 추적 | 직관적 탐색, 창의적 사고 |
| **약점** | 사람이 직접 읽기 어려움 | 기계가 의미 파악 어려움 |

---

### 이걸 안 했을 때

> 2주 전에 Inference Agent에서 성별 검출 로직을 테스트했는데, 결과가 기억나지 않았다.
> 메모리 시스템이 없었으면 "353번 이미지가 female, 349번이 male"이라는 결과를 처음부터 다시 실행해야 했다.
> Ontology에 저장되어 있었기 때문에, "Inference Agent 성별 검출 결과 알려줘"라고 물으니 즉시 나왔다.
> 반면 같은 질문을 w-company-b 봇에 하면 아무 결과도 없다 — 워크스페이스별 격리가 정확히 작동하는 것이다.

## 4.2 Ontology — 타입화된 지식 그래프

```mermaid
graph TB
    subgraph SCHEMA["Ontology Schema"]
        direction TB

        subgraph ENTITIES["Entity Types"]
            PERSON["👤 Person<br/>name, email, phone"]
            PROJECT["📁 Project<br/>name, status, goals"]
            TASK["✅ Task<br/>title, status, due, priority"]
            MESSAGE["💬 Message<br/>content, sender, timestamp"]
            THREAD["🧵 Thread<br/>subject, participants"]
            DOC["📄 Document<br/>title, path, summary"]
            ACTION["⚡ Action<br/>type, target, outcome"]
        end

        subgraph RELATIONS["Relations"]
            R1["sent_message<br/>Person → Message"]
            R2["in_thread<br/>Message → Thread"]
            R3["has_task<br/>Project → Task"]
            R4["has_owner<br/>Task → Person"]
            R5["discusses<br/>Thread → Project"]
            R6["logged_action<br/>Thread → Action"]
        end
    end

    PERSON -->|"sent_message"| MESSAGE
    MESSAGE -->|"in_thread"| THREAD
    THREAD -->|"discusses"| PROJECT
    PROJECT -->|"has_task"| TASK
    TASK -->|"has_owner"| PERSON

    style SCHEMA fill:#fafafa,stroke:#424242
    style ENTITIES fill:#e3f2fd,stroke:#1565c0
    style RELATIONS fill:#f3e5f5,stroke:#7b1fa2
```

### 제약 조건 (schema.yaml)

```mermaid
graph LR
    subgraph CONSTRAINTS["Ontology 제약 조건"]
        C1["Task.required<br/>title, status 필수"]
        C2["Task.status_enum<br/>open | in_progress<br/>blocked | done"]
        C3["Credential<br/>password, secret, token<br/>직접 저장 금지"]
        C4["Relations<br/>순환 참조 방지<br/>(acyclic)"]
    end

    style CONSTRAINTS fill:#fff3e0,stroke:#e65100
```

---

## 4.3 PARA — Obsidian 기반 지식 관리

```mermaid
graph TB
    subgraph VAULT["📚 Obsidian PARA Vault"]
        direction TB

        subgraph PROJECTS["1Projects-현재집중"]
            PJ1["company-a/<br/>Work Logs/YYYY-MM-DD.md"]
            PJ2["lgt/"]
            PJ3["company-b/"]
        end

        subgraph AREAS["2Areas-지속관리"]
            A1["개발 환경"]
            A2["인프라 관리"]
            A3["학습"]
        end

        subgraph RESOURCES["3Resources-자료"]
            RS1["Infra/<br/>서버 구축 기록"]
            RS2["Dev/<br/>기술 문서"]
        end

        subgraph ARCHIVES["4Archives-보관"]
            AR1["완료된 프로젝트"]
        end

        SYSTEM["0System/<br/>Templates/<br/>Weekly/"]
    end

    style VAULT fill:#fafafa,stroke:#424242
    style PROJECTS fill:#e8f5e9,stroke:#2e7d32
    style AREAS fill:#e3f2fd,stroke:#1565c0
    style RESOURCES fill:#fff8e1,stroke:#f9a825
    style ARCHIVES fill:#f5f5f5,stroke:#9e9e9e
```

---

## 4.4 데이터 흐름 — 수집에서 활용까지

```mermaid
graph TB
    subgraph COLLECT["1️⃣ 수집 (Ingest)"]
        SRC_TG["📱 Telegram 메시지"]
        SRC_CC["💻 Claude Code 세션"]

        ING_TG["ingest_telegram<br/>_to_ontology.py"]
        ING_CC["ingest_claude_code<br/>_to_ontology.py"]

        SRC_TG --> ING_TG
        SRC_CC --> ING_CC
    end

    subgraph STORE["2️⃣ 저장"]
        ONT["🔗 Ontology SQLite<br/>Person, Message, Thread<br/>Task, Action, Document"]
        OBS["📝 Obsidian PARA<br/>Work Logs, Weekly Recap<br/>프로젝트 노트"]

        ING_TG --> ONT
        ING_CC --> ONT
    end

    subgraph MAINTAIN["3️⃣ 유지 (Cron)"]
        M1["03:00 ontology nightly<br/>Orphan 정리, VACUUM"]
        M2["02:00 PARA dedupe<br/>중복 노트 제거"]
        M3["09:00 daily worklog<br/>Work Log 노트 생성"]
        M4["09:05 index refresh<br/>PARA 인덱스 갱신"]

        ONT --> M1 --> ONT
        OBS --> M2 --> OBS
        OBS --> M3
        OBS --> M4
    end

    subgraph USE["4️⃣ 활용"]
        AGENT["🤖 에이전트<br/>맥락 검색 + 의사결정"]
        HUMAN["👤 사용자<br/>Obsidian 앱에서 리뷰"]

        ONT -->|"ontology skill<br/>그래프 쿼리"| AGENT
        OBS -->|"iCloud 동기화<br/>아이폰/맥 어디서든"| HUMAN
    end

    style COLLECT fill:#e8eaf6,stroke:#3f51b5
    style STORE fill:#f3e5f5,stroke:#9c27b0
    style MAINTAIN fill:#fff3e0,stroke:#ef6c00
    style USE fill:#e8f5e9,stroke:#388e3c
```

---

## 4.5 Ingest 파이프라인 상세

### Telegram → Ontology

```mermaid
sequenceDiagram
    participant TG as 📱 Telegram
    participant ING as ingest_telegram<br/>_to_ontology.py
    participant ONT as 🔗 Ontology DB

    TG->>ING: 메시지 수집
    ING->>ING: 자동 분류<br/>code/dev, infra/ssh,<br/>docs/notes, finance...

    ING->>ONT: Person entity 생성<br/>person_jaechan

    ING->>ONT: Message entity 생성<br/>tg_msg_{id}

    ING->>ONT: Relation 생성<br/>person_jaechan<br/>--sent_message--><br/>tg_msg_{id}

    Note over ONT: PEM 블록 등<br/>민감정보 자동 마스킹
```

### Claude Code → Ontology

```mermaid
sequenceDiagram
    participant CC as 💻 Claude Code
    participant ING as ingest_claude_code<br/>_to_ontology.py
    participant ONT as 🔗 Ontology DB

    CC->>ING: 세션 로그 수집<br/>~/.claude/projects/**/*.jsonl

    ING->>ING: 카테고리 추출<br/>커밋, PR, 테스트,<br/>문서, obsidian, para

    ING->>ONT: Thread entity 생성<br/>세션 = 하나의 스레드

    ING->>ONT: Message entities 생성<br/>턴별 메시지

    ING->>ONT: Task entity 추출<br/>대화에서 작업 추론

    Note over ONT: Thread --discusses--> Project<br/>Thread --logged_action--> Action
```

---

### Ontology Skill 사용 예시

```mermaid
sequenceDiagram
    participant User as 📱 사용자
    participant WP as 🤖 w-company-a
    participant ONT as 🔗 w-company-a<br/>graph.sqlite

    User->>WP: "Inference Agent 관련 작업 뭐 했었지?"

    WP->>ONT: 쿼리: Project "Inference Agent"<br/>→ has_task → Task (status=*)
    ONT-->>WP: Task: 성별 검출 테스트 (done)<br/>Task: 파라미터 정규화 (done)<br/>Task: INVALID_REQUEST 400 (done)

    WP->>ONT: 쿼리: Project "Inference Agent"<br/>→ discusses ← Thread
    ONT-->>WP: Thread: 3/14 리팩토링 세션<br/>Thread: 3/15 회귀 테스트

    WP-->>User: Inference Agent 관련 작업 이력:<br/>1. 성별 검출 테스트 (3/14)<br/>2. 파라미터 정규화 (3/14~15)<br/>3. INVALID_REQUEST 400 수정 (3/15)
```

---

## 4.6 자동화와의 연결

Obsidian과 Ontology는 [5단락 스케줄러](openclaw-talk-05-scheduler.md)의 Cron 작업과 연결되어 자동으로 유지된다.

| 시간 | 작업 | 대상 |
|------|------|------|
| 02:00~02:30 | PARA vault 중복 제거 | Obsidian |
| 03:00 | nightly maintenance (Orphan 정리, VACUUM) | Ontology |
| 09:00~09:10 | Work Log / 인덱스 / Weekly Recap 자동 생성 | Obsidian |

> 수집 → 저장 → 유지 → 활용의 전체 사이클이 사람 개입 없이 돌아간다.

---

## 4.7 핵심 가치

> **"AI는 그래프를 쿼리하고, 사람은 노트를 읽는다."**
>
> Ontology는 에이전트가 **구조화된 맥락을 빠르게 검색**할 수 있게 하고,
> Obsidian PARA는 사용자가 **직관적으로 지식을 탐색**할 수 있게 한다.
> 두 저장소가 함께 동작하면서, AI와 사람 모두에게 최적화된 메모리를 제공한다.

```mermaid
graph LR
    subgraph CYCLE["지식 순환 사이클"]
        COLLECT["📥 수집<br/>Telegram, Claude Code"]
        STRUCTURE["🔗 구조화<br/>Ontology<br/>(Entity → Relation)"]
        HUMANIZE["📝 인간화<br/>Obsidian PARA<br/>(마크다운 노트)"]
        MAINTAIN_C["🔧 유지<br/>야간 Cron<br/>(정리, 인덱싱)"]
        UTILIZE["💡 활용<br/>AI 쿼리 + 사람 리뷰"]
    end

    COLLECT --> STRUCTURE --> HUMANIZE --> MAINTAIN_C --> UTILIZE --> COLLECT

    style CYCLE fill:#fafafa,stroke:#424242
```

| 기존 방식 | Ontology + PARA |
|-----------|----------------|
| 대화 기록만 남김 | 구조화된 그래프로 관계 추적 |
| AI가 과거 맥락 모름 | ontology skill로 즉시 쿼리 |
| 노트 정리는 수동 | Cron이 매일 자동 생성/정리 |
| 프로젝트 간 지식 혼재 | 워크스페이스별 DB 격리 |
| 사람 또는 기계 한쪽만 최적 | 양쪽 모두에 최적화 |

---

*다음 단락: 5. 스케줄러*
