# 내가 경험한 OpenClaw — 1. 개발 환경

> **SSH Key 인증 기반 멀티 서버 개발 환경, 그리고 OpenClaw가 만드는 생산성**

---

## 1.1 전체 개발 환경 구조

```mermaid
graph TB
    subgraph LOCAL["🖥️ Local Mac (Client)"]
        IDE["VS Code / Terminal"]
        OC["OpenClaw Gateway<br/>(localhost:18789)"]
        TG["Telegram Bot<br/>(6개 에이전트)"]
    end

    subgraph DC_A["🏢 Data Center A"]
        WEB243["dc-a-243-webserver<br/>Project D MSA API"]
        GPU247["dc-a-247-gpu05<br/>AI 연산"]
    end

    subgraph COMPANY_A["🏢 Company A Dev"]
        PWEB["company-a-dev-webserver<br/>Web Service"]
        GPU1["company-a-dev-gpu1<br/>GPU #1"]
        GPU2["company-a-dev-gpu2<br/>GPU #2"]
    end

    subgraph HOME["🏠 자택 인프라"]
        MM["macmini-main"]
        MS["macmini-sub"]
        WSL["WSL-01"]
    end

    IDE -->|"SSH ed25519"| DC_A
    IDE -->|"SSH ed25519"| COMPANY_A
    IDE -->|"SSH ed25519"| HOME
    OC -->|"SSH 통해 원격 제어"| DC_A
    OC -->|"SSH 통해 원격 제어"| COMPANY_A
    OC -->|"SSH 통해 원격 제어"| HOME
```

---

## 1.2 SSH Config가 만드는 생산성

17개 서버, 포트도 유저도 키도 제각각이다. 그래도 접속은 **한 줄**이면 끝난다.

```mermaid
graph LR
    subgraph BEFORE["❌ SSH Config 없이"]
        B1["ssh -i ~/.ssh/id_ed25519_dc_a_243_webserver<br/>-p 8703 company-a@203.0.113.243"]
    end

    subgraph AFTER["✅ SSH Config 적용"]
        A1["ssh dc-a-243-webserver"]
    end

    BEFORE -.->|"매번 기억? 불가능"| AFTER
```

### 이걸 안 했을 때

> SSH config 정리 전에는 서버 접속 자체가 스트레스였다.
> 포트가 8703인지 22인지 헷갈려서 타임아웃 대기, 키를 잘못 지정해서 인증 실패,
> 심지어 프로덕션 서버에 dev 계정으로 접속하는 실수도 있었다.
> 17개 서버를 머릿속에 담고 다니는 건 불가능하다. alias 하나면 끝나는 일이었다.

### SSH Config 설계 원칙

| 원칙 | 적용 | 효과 |
|------|------|------|
| **서버별 전용 키 분리** | `id_ed25519_dc_a_243_webserver` 등 11개 키 | 키 유출 시 blast radius 최소화 |
| **직관적 alias** | `dc-a-243-webserver`, `company-b-us-ec2-production` | 용도가 이름에 드러남 |
| **Keychain 연동** | `AddKeysToAgent yes` + `UseKeychain yes` | 패스프레이즈 반복 입력 제거 |
| **SSM ProxyCommand** | AWS EC2는 포트 노출 없이 터널링 | 보안과 편의성 동시 확보 |

---

## 1.3 핵심: 로컬에서 떠나지 않고 원격을 제어한다

```mermaid
sequenceDiagram
    participant Dev as 🖥️ Local Mac
    participant SSH as SSH Tunnel
    participant Server as 🏢 Data Center A<br/>dc-a-243-webserver
    participant Docker as 🐳 Docker Container<br/>inference-api

    Note over Dev: 로컬에서 코드 작성 중...

    Dev->>SSH: ssh dc-a-243-webserver
    SSH->>Server: 접속 (ed25519, port 8703)

    rect rgb(230, 245, 255)
        Note over Dev,Docker: 원격 코드 읽기 & 수정
        Dev->>Server: cat src/application/job_service.py
        Server-->>Dev: 코드 내용 확인
        Dev->>Server: sed / cat > 으로 코드 수정
    end

    rect rgb(255, 245, 230)
        Note over Dev,Docker: Docker 컨테이너 내부 실행
        Dev->>Server: docker exec -i inference-api python -
        Server->>Docker: 성별 검출기 실행
        Docker-->>Dev: female / male 결과 반환
    end

    rect rgb(230, 255, 230)
        Note over Dev,Docker: API 테스트 & 배포
        Dev->>Server: curl localhost:18031/api/services
        Server-->>Dev: 서비스 목록 응답
        Dev->>Server: docker restart inference-api
        Server->>Docker: 컨테이너 재시작
        Dev->>Server: health check 확인
        Server-->>Dev: ✅ healthy
    end

    rect rgb(245, 230, 255)
        Note over Dev,Server: 커밋 & 회귀 테스트
        Dev->>Server: git add . && git commit
        Dev->>Server: python scripts/regression_smoke.py
        Server-->>Dev: 5개 서비스 ✅ PASSED
    end
```

---

## 1.4 실제 작업 사례: Project D Kiosk ↔ Inference Agent API

2026년 3월 14~15일, 약 4시간 집중 작업의 실제 흐름.

```mermaid
graph TB
    subgraph LOCAL["🖥️ Local Mac"]
        KIOSK["Kiosk Electron App<br/>(kiosk-electron/)"]
        TTS["TTS 캐시 로직 수정<br/>contentHubTtsClient.ts<br/>ttsPublicCache.ts<br/>ttsHandlers.ts"]
        ONTO["Ontology DB<br/>프로젝트 메타 저장"]
    end

    subgraph REMOTE["🏢 Data Center A (SSH)"]
        PBA["Inference Agent FastAPI<br/>inference-agent-fastapi/"]
        DOCKER["Docker Container<br/>inference-api:18031"]
        GIT["Git Repository<br/>원격 직접 커밋"]
    end

    KIOSK -->|"로컬 코드 편집<br/>8개 파일 수정"| TTS
    TTS -.->|"TTS API 호출 흐름"| PBA

    PBA -->|"SSH로 코드 수정<br/>10+ 파일 리팩토링"| DOCKER
    DOCKER -->|"docker restart<br/>→ health check"| PBA
    PBA -->|"회귀 테스트<br/>5개 서비스 검증"| GIT
    GIT -->|"12개 커밋"| PBA

    LOCAL <-->|"SSH ed25519<br/>port 8703"| REMOTE

    style LOCAL fill:#e8f4f8,stroke:#2196F3
    style REMOTE fill:#fff3e0,stroke:#FF9800
```

### 작업 타임라인

```mermaid
gantt
    title 3/14~15 Project D Kiosk ↔ Inference Agent 작업 흐름
    dateFormat HH:mm
    axisFormat %H:%M

    section 로컬 (Kiosk)
    TTS 캐시 로직 분석 & 수정 (8파일)       :done, 18:50, 19:19

    section 원격 (Data Center A)
    SSH 접속 확인 (포트 8703)                :done, 17:59, 18:04
    Inference Agent 코드 분석 & 성별 검출 테스트     :done, 18:04, 18:45
    프리셋 추가 & API 설정 업데이트           :done, 19:07, 19:32
    코드 리팩토링 (10+ 파일)                 :done, 21:35, 23:30
    회귀 테스트 & 커밋 (12개)                :done, 23:30, 25:51
```

---

## 1.5 OpenClaw가 이 환경에서 하는 일

```mermaid
graph LR
    subgraph TELEGRAM["📱 Telegram"]
        USER["사용자<br/>(어디서든)"]
    end

    subgraph OPENCLAW["🤖 OpenClaw Gateway"]
        BOT["w-company-a4sky Bot"]
        AGENT["AI Agent<br/>(gpt-5.2)"]
    end

    subgraph SERVERS["🖥️ SSH 연결된 서버들"]
        S1["company-prod : API 서버"]
        S2["company-dev : 웹서버"]
        S3["macmini : home 인프라"]
    end

    USER -->|"메시지"| BOT
    BOT --> AGENT
    AGENT -->|"SSH로 코드 읽기"| S2
    AGENT -->|"SSH로 로그 확인"| S2
    AGENT -->|"SSH로 Docker 제어"| S2
    AGENT -->|"SSH로 git commit"| S2
    AGENT -->|"SSH로 서버 관리"| S1
    AGENT -->|"SSH로 서비스 모니터링"| S3
    S1 -->|"결과"| AGENT
    AGENT -->|"응답"| USER

    style TELEGRAM fill:#e3f2fd
    style OPENCLAW fill:#f3e5f5
    style SERVERS fill:#e8f5e9
```

### 핵심 가치

> **"내 Mac에서 떠나지 않는다."**
>
> 코드는 로컬에서 작성하고, SSH 하나로 원격 서버의 코드·로그·Docker·배포를 실시간 제어한다.
> OpenClaw는 이 SSH 인프라 위에서 Telegram을 통해 **어디서든** 같은 작업을 가능하게 한다.

| 전통적 방식 | SSH + OpenClaw |
|------------|----------------|
| 서버마다 접속 정보 기억 | alias 한 줄로 즉시 접속 |
| 서버에서 직접 작업 | 로컬에서 원격 실시간 제어 |
| 터미널 앞에 있어야 함 | Telegram으로 어디서든 제어 |
| 수동 배포 & 확인 | SSH → Docker restart → 자동 회귀 테스트 |
| 키 하나로 전체 접속 | 서버별 전용 키 분리 (보안) |

---

*다음 단락: 2. OpenClaw 에이전트 구성과 운영*
