# 내가 경험한 OpenClaw — 6. Amphetamine

> **MacBook을 닫아도, 이동 중에도 — 대화는 끊기지 않는다**

*지금까지 설명한 모든 것 — 멀티 에이전트, Runbook, Obsidian/Ontology, 야간 스케줄러 — 의 전제가 하나 있다.
Mac이 꺼지지 않아야 한다.*

---

## 6.1 문제: Mac이 잠들면 모든 게 멈춘다

```mermaid
graph TB
    subgraph PROBLEM["❌ 기본 macOS 동작"]
        direction TB
        P1["MacBook 덮개 닫기<br/>또는 10분 유휴"]
        P2["macOS 잠자기 진입"]
        P3["모든 프로세스 정지"]

        P1 --> P2 --> P3

        subgraph STOPPED["멈추는 것들"]
            S1["🤖 OpenClaw Gateway<br/>Telegram 응답 불가"]
            S2["⏰ Cron 작업<br/>야간 자동화 중단"]
            S3["🔌 SSH 연결<br/>원격 세션 끊김"]
            S4["📡 네트워크<br/>핫스팟 연결 해제"]
        end

        P3 --> STOPPED
    end

    style PROBLEM fill:#fff0f0,stroke:#e53935
    style STOPPED fill:#ffcdd2,stroke:#c62828
```

OpenClaw Gateway는 **LaunchAgent로 상시 구동**(KeepAlive: true)되지만, Mac이 잠들면 의미가 없다.

### 이걸 안 했을 때

> Amphetamine을 설정하기 전, 밤에 Mac 덮개를 닫고 잤다.
> 아침에 Telegram을 열었는데 리포트가 하나도 와 있지 않았다. 야간 Cron 51개 전부 실패.
> 코드리뷰도, PARA 정리도, 야간 개발 검토도 — 전부 멈춰 있었다.
> Mac이 잠들지 않게 하는 것, 이 하나가 전체 자동화의 전제 조건이었다.

---

## 6.2 해결: Amphetamine으로 잠자기 방지

```mermaid
graph TB
    subgraph SOLUTION["✅ Amphetamine 적용"]
        direction TB
        AMP["💊 Amphetamine<br/>macOS 잠자기 방지 앱"]

        subgraph SETTINGS["현재 설정"]
            SET1["Default Duration: 무제한 (Indefinite)"]
            SET2["Allow Display Sleep: Off<br/>디스플레이도 깨어있음"]
            SET3["Allow Screen Saver: Off"]
            SET4["Hide Dock Icon: On<br/>메뉴바에서만 동작"]
        end

        subgraph RESULT["결과"]
            R1["🤖 OpenClaw 24/7 상시 구동"]
            R2["⏰ Cron 작업 무중단 실행"]
            R3["🔌 SSH 연결 유지"]
            R4["📡 네트워크 연결 유지"]
        end

        AMP --> SETTINGS --> RESULT
    end

    style SOLUTION fill:#f0fff0,stroke:#43a047
    style SETTINGS fill:#e8f5e9,stroke:#2e7d32
    style RESULT fill:#c8e6c9,stroke:#1b5e20
```

### pmset 현재 상태

```
sleep: 0 (sleep prevented by coreaudiod, powerd, caffeinate)
displaysleep: 10
tcpkeepalive: 1
powernap: 1
womp: 1 (Wake on LAN)
```

**sleep: 0** — Amphetamine이 caffeinate를 통해 시스템 잠자기를 완전 차단.

---

## 6.3 이동 중 워크플로우: 핫스팟 + Telegram

```mermaid
sequenceDiagram
    participant User as 📱 사용자<br/>(이동 중)
    participant Phone as 📶 iPhone<br/>핫스팟
    participant Mac as 💻 MacBook Air M4<br/>(가방 안)
    participant OC as 🤖 OpenClaw<br/>Gateway
    participant TG as 📬 Telegram<br/>Bot

    Note over User,TG: 🚶 이동 시작 — 핫스팟 연결

    User->>Phone: Wi-Fi 핫스팟 활성화
    Phone->>Mac: MacBook 자동 연결
    Note over Mac: Amphetamine: 잠자기 방지 중<br/>sleep: 0

    rect rgb(230, 245, 255)
        Note over User,TG: 이동 중에도 대화 계속
        User->>TG: @company-a4sky_bot<br/>"Inference Agent 로그 확인해줘"
        TG->>OC: 메시지 수신 (polling)
        OC->>OC: w-company-a 에이전트 실행
        OC->>TG: SSH → 로그 확인 → 결과 응답
        TG->>User: 📱 결과 수신
    end

    rect rgb(255, 245, 230)
        Note over User,TG: 다른 프로젝트로 전환
        User->>TG: @company-b4sky_bot<br/>"Products 상태 확인해줘"
        TG->>OC: w-company-b 에이전트 라우팅
        OC->>TG: 결과 응답
        TG->>User: 📱 결과 수신
    end

    Note over User,TG: 🏢 도착 — 동일한 세션 이어서 작업
```

---

## 6.4 항상 켜져 있는 인프라 스택

```mermaid
graph TB
    subgraph STACK["💻 MacBook Air M4 — Always-On Stack"]
        direction TB

        subgraph LAYER1["OS 레이어"]
            AMP["💊 Amphetamine<br/>잠자기 방지"]
            LAUNCH["LaunchAgent<br/>KeepAlive: true"]
            PMSET["pmset sleep: 0<br/>tcpkeepalive: 1"]
        end

        subgraph LAYER2["서비스 레이어"]
            OC_GW["🤖 OpenClaw Gateway<br/>localhost:18789"]
            OC_BOT["📬 Telegram Bots × 6<br/>polling 상시 활성"]
            OC_CRON["⏰ Cron Engine<br/>51개 작업"]
        end

        subgraph LAYER3["네트워크 레이어"]
            WIFI["📶 Wi-Fi<br/>자택: 고정 AP<br/>이동: iPhone 핫스팟"]
            SSH_OUT["🔌 SSH Connections<br/>17개 원격 서버"]
            TG_API["📡 Telegram API<br/>Bot polling"]
        end
    end

    AMP -->|"caffeinate"| PMSET
    LAUNCH -->|"KeepAlive"| OC_GW
    OC_GW --> OC_BOT
    OC_GW --> OC_CRON
    WIFI --> SSH_OUT
    WIFI --> TG_API
    OC_BOT --> TG_API
    OC_CRON --> SSH_OUT

    style STACK fill:#fafafa,stroke:#424242
    style LAYER1 fill:#fff3e0,stroke:#ef6c00
    style LAYER2 fill:#f3e5f5,stroke:#9c27b0
    style LAYER3 fill:#e3f2fd,stroke:#1565c0
```

---

## 6.5 덮개를 닫아도 동작하는 이유

```mermaid
graph TB
    subgraph CLOSED["MacBook 덮개 닫힘"]
        direction TB

        subgraph WITHOUT_AMP["❌ Amphetamine 없이"]
            C1["덮개 닫기"]
            C2["잠자기 진입"]
            C3["프로세스 정지"]
            C1 --> C2 --> C3
        end

        subgraph WITH_AMP["✅ Amphetamine + 외부 전원"]
            A1["덮개 닫기"]
            A2["Amphetamine이<br/>caffeinate 유지"]
            A3["Display만 꺼짐<br/>시스템은 깨어있음"]
            A4["OpenClaw 계속 동작<br/>SSH 유지, Cron 실행"]
            A1 --> A2 --> A3 --> A4
        end
    end

    style WITHOUT_AMP fill:#ffcdd2,stroke:#c62828
    style WITH_AMP fill:#c8e6c9,stroke:#2e7d32
```

### 핵심 조건

| 조건 | 역할 |
|------|------|
| **Amphetamine** | caffeinate로 시스템 잠자기 차단 |
| **전원 연결** (자택/사무실) | 배터리 걱정 없이 상시 구동 |
| **핫스팟** (이동 중) | 네트워크 연결 유지 |
| **tcpkeepalive: 1** | TCP 연결 끊김 방지 |
| **womp: 1** | Wake on LAN 지원 |
| **KeepAlive: true** | Gateway 크래시 시 자동 재시작 |

---

## 6.6 핵심 가치

> **"장소가 바뀌어도, Mac 덮개를 닫아도, 대화는 끊기지 않는다."**
>
> Amphetamine 하나로 MacBook Air M4가 **항상 켜진 서버**가 된다.
> 핫스팟만 연결하면 이동 중에도 Telegram으로 6개 에이전트와 대화하고,
> 자는 동안에는 51개 Cron 작업이 멈추지 않는다.

```mermaid
graph LR
    AMP["💊 Amphetamine<br/>잠자기 방지"]
    MAC["💻 MacBook<br/>= Always-On 서버"]
    OC["🤖 OpenClaw<br/>24/7 무중단"]
    USER["📱 사용자<br/>언제 어디서든"]

    AMP -->|"caffeinate"| MAC
    MAC -->|"LaunchAgent"| OC
    OC -->|"Telegram"| USER

    style AMP fill:#fff3e0,stroke:#ef6c00
    style MAC fill:#e3f2fd,stroke:#1565c0
    style OC fill:#f3e5f5,stroke:#9c27b0
    style USER fill:#e8f5e9,stroke:#388e3c
```

| 일반 MacBook 사용 | Amphetamine + OpenClaw |
|-------------------|----------------------|
| 덮개 닫으면 잠자기 | 덮개 닫아도 시스템 깨어있음 |
| 이동하면 작업 중단 | 핫스팟으로 이동 중 대화 계속 |
| 밤에 Mac 꺼짐 | 야간 자동화 무중단 실행 |
| 크래시 시 수동 재시작 | KeepAlive로 자동 복구 |
| 장소마다 환경 재설정 | 어디서든 동일한 에이전트 접근 |

---

*다음 단락: 7. 자동 에러 감지 & 수정*
