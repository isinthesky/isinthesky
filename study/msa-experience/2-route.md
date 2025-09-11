# MSA 환경에서 Nginx 리버스 프록시 API 라우팅 규칙

## 문제 정의

MSA 환경으로 전환하면서 가장 먼저 직면한 문제는 **클라이언트가 여러 개의 독립적인 서비스에 어떻게 접근할 것인가**였습니다.

### Monolith 환경의 단순함

- 클라이언트 → 단일 서버 (모든 API 통합)
- 하나의 도메인, 하나의 포트
- 내부 함수 호출로 서비스 간 통신

### MSA 환경의 복잡성

- 클라이언트 → 5개의 독립 서비스
- 각각 다른 포트와 주소
- 클라이언트가 모든 서비스 위치를 알아야 함

**결론**: Nginx 리버스 프록시가 필요하다!

## API 라우팅 아키텍처

### 1. 전체 라우팅 구조

```mermaid
graph TB
    subgraph "클라이언트 레이어"
        Web[Web App]
        Mobile[Mobile App]
        External[External API 호출]
    end
    
    subgraph "Nginx 리버스 프록시"
        Gateway[Nginx<br/>리버스 프록시<br/>Port: 80/443]
        
        subgraph "라우팅 규칙"
            Route1["/api/v1/main/*<br/>→ Main API"]
            Route2["/api/v1/auth/*<br/>→ Auth Service"]
            Route3["/api/v1/notification/*<br/>→ Notification Service"]
            Route4["/api/v1/storage/*<br/>→ Storage Service"]
            Route5["/api/v1/ai/*<br/>→ AI GPU Service"]
        end
    end
    
    subgraph "MSA 서비스들"
        MainAPI[Main API Server<br/>:3000]
        AuthService[Auth Server<br/>:3001]
        NotiService[Notification Server<br/>:3002]
        StorageService[Storage Server<br/>:3003]
        AIService[AI GPU Server<br/>:3004]
    end
    
    Web --> Gateway
    Mobile --> Gateway
    External --> Gateway
    
    Gateway --> Route1 --> MainAPI
    Gateway --> Route2 --> AuthService
    Gateway --> Route3 --> NotiService
    Gateway --> Route4 --> StorageService
    Gateway --> Route5 --> AIService
    
    style Gateway fill:#ffebee
    style Route1 fill:#e8f5e9
    style Route2 fill:#fff3e0
    style Route3 fill:#f3e5f5
    style Route4 fill:#e0f2f1
    style Route5 fill:#fce4ec
```

## 라우팅 규칙 상세 설계

### 서비스별 URL 패턴

| 서비스 | URL 패턴 | 역할 | api url 예시 | back office page url  예시
|--------|----------|------|------|---|
| **Main API** | `/api/v1/main/*` | 핵심 비즈니스 로직 | `/api/v1/main/products`, `/api/v1/main/processing` | `/page/main/dashboard`, `/page/main/logs/` |
| **Auth** | `/api/v1/auth/*` | 인증/인가 | `/api/v1/auth/login`, `/api/v1/auth/users` | `/page/auth/users`, `/page/auth/roles` |
| **Notification** | `/api/v1/notification/*` | 알림 처리 | `/api/v1/notification/send`, `/api/v1/notification/history` | `/page/notification/templates`, `/page/notification/logs` |
| **Storage** | `/api/v1/storage/*` | 파일 관리 | `/api/v1/storage/upload`, `/api/v1/storage/download` | `/page/storage/files`, `/page/storage/quota` |
| **AI** | `/api/v1/ai/*` | AI 연산 | `/api/v1/ai/predict`, `/api/v1/ai/train` | `/page/ai/models`, `/page/ai/metrics` |

### 3. 로드밸런싱 전략

```mermaid
graph TB
    subgraph "로드밸런싱 구조"
        Client[클라이언트 요청]
        Nginx[Nginx Load Balancer]
        
        subgraph "Storage Cluster"
            STORAGE1[Storage 1<br/>:3031]
            STORAGE2[Storage 2<br/>:3032]
            STORAGE3[Storage 3<br/>:3033]
        end
        
        subgraph "AI GPU Cluster"
            AI1[AI GPU 1<br/>:3041]
            AI2[AI GPU 2<br/>:3042]
            AI3[AI GPU 3<br/>:3043]
        end
        
        subgraph "Single Instance Services"
            MAIN[Main API Server<br/>:3011]
            AUTH[Auth Server<br/>:3021]
            NOTI[Notification Server<br/>:3051]
        end
    end
    
    Client --> Nginx
    Nginx --> STORAGE1
    Nginx --> STORAGE2
    Nginx --> STORAGE3
    Nginx --> AI1
    Nginx --> AI2
    Nginx --> AI3
    Nginx --> MAIN
    Nginx --> AUTH
    Nginx --> NOTI
    
    style Client fill:#e1f5fe
    style Nginx fill:#ffebee
    style STORAGE1 fill:#c8e6c9
    style STORAGE2 fill:#c8e6c9
    style STORAGE3 fill:#c8e6c9
    style AI1 fill:#e1bee7
    style AI2 fill:#e1bee7
    style AI3 fill:#e1bee7
    style MAIN fill:#fff3e0
    style AUTH fill:#f3e5f5
    style NOTI fill:#e0f2f1
```

## 추천하는 라우팅 규칙 설계 원칙

### 1. 명확한 서비스 경계

- 각 서비스는 고유한 URL prefix 사용
- `/api/v1`은 핵심 비즈니스만, 인프라성 기능은 별도 경로

### 2. 버전 관리 전략

- API 버전은 URL에 명시적으로 포함
- 하위 호환성을 위한 점진적 마이그레이션

### 3. 확장성 고려

- 새로운 서비스 추가 시 기존 규칙에 영향 없도록
- 마이크로서비스의 독립성 보장

### 4. 보안 고려사항

- 민감한 API는 별도 인증 레이어 적용
- Rate limiting으로 서비스별 트래픽 제어

이러한 라우팅 규칙을 통해 클라이언트는 단일 진입점을 통해 모든 MSA 서비스에 접근할 수 있으며, 운영팀은 중앙집중식 트래픽 관리가 가능해집니다.