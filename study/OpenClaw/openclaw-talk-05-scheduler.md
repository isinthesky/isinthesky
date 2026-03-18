# 내가 경험한 OpenClaw — 5. 스케줄러

> **내가 자는 동안에도 작업이 돌아간다**

*Obsidian과 Ontology라는 저장소가 갖춰졌다. 이제 여기에 데이터를 자동으로 채워야 한다.
코드리뷰, 데이터 정리, 아침 리포트 — 자는 동안에도 에이전트가 일하게 만들고 싶었다.
그래서 OpenClaw의 Cron 스케줄러를 설계했다.*

---

## 5.1 전체 스케줄 타임라인

```mermaid
gantt
    title 밤새 AI가 일하고, 아침에 리포트를 받는다 (KST)
    dateFormat HH:mm
    axisFormat %H:%M

    section 🌙 야간 자동화
    PARA vault 중복 제거               :para, 01:00, 1h
    Ontology 유지보수                  :onto, 02:00, 1h
    데스크톱 파일 정리 (Vision AI)       :desk, 03:00, 1h
    A-1 코드리뷰 + 야간 개발 검토       :cr1, 04:00, 1h
    A-b 코드리뷰 + 야간 개발 검토       :cr2, 05:00, 1h
    CompanyB 코드리뷰 + 야간 개발 검토  :cr3, 06:00, 1h
    야간 작업 결과 취합                 :agg, 07:00, 1h

    section ☀️ 아침 리포트
    Telegram 종합 리포트 전송           :crit, rpt, 08:00, 30min

    section 🧑‍💻 업무 시간
    사용자 대화 기반 작업               :manual, 10:00, 8h

    section 😴 취침
    에이전트 자율 작업 대기              :sleep, 23:00, 2h
```

---

### 이걸 안 했을 때

> 스케줄러 도입 전에는, 아침에 출근하면 각 서버 로그를 하나씩 열어보는 것부터 시작했다.
> Data Center A 서버 접속 → docker logs 확인 → Company B EC2 접속 → 배포 상태 확인 → Project C DB 상태 확인.
> 매일 1시간이 소요됐다. 지금은 Telegram에 아침 리포트가 와 있고, 5분이면 전체 상황 파악이 끝난다.
> 시간도 아꼈지만, 더 좋았던 건 **인지 부하가 줄었다**는 점이다. "혹시 어젯밤에 뭐 터졌나?" 같은 불안이 사라졌다.

## 5.2 자동화 구조

```mermaid
graph TB
    subgraph CRON["⏰ OpenClaw Cron Engine"]
        JOBS["jobs.json<br/>51개 작업 정의"]
        SCHED["Scheduler<br/>Asia/Seoul TZ"]
        LANE["Lane System<br/>큐 관리 + 동시성 제어"]
    end

    subgraph AGENTS["🤖 에이전트별 분배"]
        MAIN["main<br/>중앙 자동화<br/>170개 세션"]
        WP["w-company-a<br/>개발 검토<br/>68개 세션"]
        WC["w-company-b<br/>보안 검사<br/>10개 세션"]
        WL["w-lgt<br/>개발 검토<br/>8개 세션"]
        PL["planner<br/>계획<br/>2개 세션"]
    end

    subgraph DELIVERY["📬 결과 전달"]
        TG["Telegram<br/>아침 리포트"]
        FILE["파일 시스템<br/>로그 + 정리 결과"]
        OBS["Obsidian<br/>노트 자동 생성"]
    end

    JOBS --> SCHED --> LANE
    LANE --> MAIN
    LANE --> WP
    LANE --> WC
    LANE --> WL
    LANE --> PL

    MAIN --> TG
    MAIN --> OBS
    WP --> TG
    WP --> FILE
    WC --> TG
    WL --> TG

    style CRON fill:#f3e5f5,stroke:#9c27b0
    style AGENTS fill:#e3f2fd,stroke:#1976d2
    style DELIVERY fill:#e8f5e9,stroke:#388e3c
```

---

## 5.3 수면 시간 자동화 상세

내가 자는 동안(약 23:30 ~ 08:00) 실행되는 작업들.

```mermaid
graph TB
    subgraph SLEEP_WINDOW["😴 수면 시간 자동화"]
        direction TB

        subgraph FILE_MGMT["📁 파일 정리"]
            F1["02:00 PARA 1Projects 중복 제거"]
            F2["02:30 PARA 2Areas 중복 제거"]
            F3["03:30 데스크톱 파일 자동 정리<br/>(Vision AI 분류)"]
        end

        subgraph DEV_REVIEW["🔍 야간 개발 검토"]
            D1["03:30 Company A night dev checks"]
            D2["03:40 Company C night dev checks"]
            D3["03:50 Company B night dev checks"]
            D4["00:00~07:00 Company A APIs<br/>30분 간격 코드리뷰"]
        end

        subgraph DATA["📊 데이터 & 지식"]
            K1["03:00 Ontology 유지보수"]
            K2["03:35 수면 중 ingest + pre-review"]
        end
    end

    style SLEEP_WINDOW fill:#e8eaf6,stroke:#3f51b5
    style FILE_MGMT fill:#fff3e0,stroke:#ef6c00
    style DEV_REVIEW fill:#e3f2fd,stroke:#1565c0
    style DATA fill:#f3e5f5,stroke:#7b1fa2
```

### 야간 개발 검토 — 30분 간격 코드리뷰

```mermaid
sequenceDiagram
    participant CRON as ⏰ Cron
    participant WP as 🤖 w-company-a
    participant SERVER as 🏢 Data Center A

    loop 매 30분 (00:00 ~ 07:00)
        CRON->>WP: Company A APIs 코드리뷰 실행
        WP->>SERVER: SSH → 코드 변경 확인
        WP->>WP: 애매함(ambiguity) 구체화
        WP->>WP: 개선 포인트 기록

        alt 이슈 발견
            WP->>WP: 메모리에 저장
            Note over WP: 08:30 아침 리포트에 포함
        end
    end

    Note over CRON,SERVER: 자는 동안 14회 자동 코드리뷰
```

---

## 5.4 아침 리포트 — 잠에서 깨면 Telegram에

```mermaid
sequenceDiagram
    participant CRON as ⏰ Cron
    participant MAIN as 🤖 main
    participant WP as 🤖 w-company-a
    participant WL as 🤖 w-lgt
    participant WC as 🤖 w-company-b
    participant TG as 📱 Telegram

    Note over CRON,TG: 08:00~09:30 아침 리포트 시퀀스

    CRON->>MAIN: 08:00 야간 작업 완료 확인
    MAIN->>MAIN: 야간 작업 결과 점검

    par 08:30~32 에이전트별 리포트
        CRON->>WP: 08:30 Company A 리포트
        WP->>TG: 야간 코드리뷰 결과 + 이슈 요약
        CRON->>WL: 08:31 Company C 리포트
        WL->>TG: 야간 개발 검토 결과
        CRON->>WC: 08:32 Company B 리포트
        WC->>TG: 보안 검사 + 상태 요약
    end

    CRON->>MAIN: 08:50 일일 다이제스트
    MAIN->>TG: 수면 후 종합 요약

    CRON->>MAIN: 09:00 PARA 워크로그
    MAIN->>TG: Obsidian 일일 워크로그

    CRON->>MAIN: 09:30 야간 스케줄러 요약
    MAIN->>TG: 전체 cron 실행 결과 보고

    Note over TG: 📱 아침에 Telegram 열면<br/>밤새 일어난 일이 정리되어 있음
```
---

## 5.5 핵심 가치

> **"자는 동안에도 에이전트는 일한다."**
>
> cron 작업이 파일 정리, 코드 리뷰, 데이터 유지보수를 수행하고,
> 아침에 Telegram을 열면 밤새 일어난 모든 일이 요약되어 기다리고 있다.

| 내가 자는 시간 | AI가 하는 일 |
|----------------|-------------|
| 00:00 ~ 02:00 | Company A API 코드리뷰 (30분 간격) |
| 02:00 ~ 03:00 | PARA vault 중복 제거 |
| 03:00 ~ 04:00 | Ontology 유지보수, 데스크톱 정리, 3개 프로젝트 야간 개발 검토 |
| 05:00 ~ 07:00 | 마감 코드리뷰 |
| 08:00 ~ 09:30 | 완료 확인 → 에이전트별 리포트 → 종합 요약 → Telegram 전송 |

```mermaid
graph LR
    SLEEP["😴 수면<br/>23:30~08:00"]
    WORK["🤖 자동화 작업 실행<br/>코드리뷰 / 파일정리<br/>데이터유지"]
    WAKE["☀️ 기상<br/>Telegram 확인"]
    REPORT["📋 아침 리포트<br/>밤새 일어난 일 요약"]

    SLEEP -->|"자는 동안"| WORK
    WORK -->|"08:00~09:30"| REPORT
    REPORT -->|"Telegram"| WAKE

    style SLEEP fill:#e8eaf6,stroke:#3f51b5
    style WORK fill:#f3e5f5,stroke:#9c27b0
    style REPORT fill:#fff8e1,stroke:#f9a825
    style WAKE fill:#e8f5e9,stroke:#388e3c
```

---

*다음 단락: 6. Amphetamine*
