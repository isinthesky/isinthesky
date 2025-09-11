# MSA 환경에서 지연 실행 AI 이미지 처리 워크플로우

## 개요

사용자가 AI 이미지 처리 작업을 8시간 후에 실행되도록 스케줄을 등록했을 때, MSA 환경에서 발생하는 API Flow와 Image Data Flow를 상세히 분석합니다.

## 시나리오 정의

- **작업**: AI 이미지 처리 (배경 제거, 스타일 변환 등)
- **지연 시간**: 8시간 후 실행
- **이미지 크기**: 10MB
- **처리 시간**: 약 5분 예상

## 1. 스케줄 등록 단계 (Initial API Flow)

### API 요청 플로우

```mermaid
sequenceDiagram
    participant User as 사용자
    participant WebApp as 웹 애플리케이션
    participant MainAPI as Main API Server
    participant AuthAPI as Auth Service
    participant StorageAPI as Storage Service
    participant AIAPI as AI Service & Task Queue
    
    User->>WebApp: 이미지 업로드 + 8시간 지연 설정
    WebApp->>MainAPI: POST /api/v1/main/ai-tasks<br/>Cookie Token + FormData
    
    MainAPI->>MainAPI: Cookie Token 검증
    MainAPI->>AuthAPI: GET /api/v1/auth/validate<br/>Header Token (변환)
    AuthAPI->>MainAPI: 사용자 권한 정보
    
    MainAPI->>StorageAPI: POST /api/v1/storage/upload<br/>Header Token + Image
    StorageAPI->>StorageAPI: 이미지 저장 (S3/Local)
    StorageAPI->>MainAPI: {"file_id": "img_123", "url": "https://..."}
    
    MainAPI->>AIAPI: POST /api/v1/ai/schedule<br/>Header Token + {"file_id", "delay": "8h"}
    AIAPI->>AIAPI: 스케줄 작업 저장 + 큐 등록<br/>{"user_id", "file_id", "delay", "status"}
    AIAPI->>MainAPI: {"task_id": "task_456", "status": "scheduled"}
    
    MainAPI->>WebApp: {"task_id": "task_456", "status": "scheduled"}
    WebApp->>User: 스케줄 등록 완료 알림
```

## 2. 스케줄러 동작 (8시간 대기)

### Task Queue 관리

```mermaid
graph LR
    subgraph "Task Queue 시스템"
        subgraph "등록 시점 (T+0)"
            Queue1[Task Queue<br/>task_456<br/>delay: 8h<br/>status: pending]
        end
        
        subgraph "대기 중 (T+1~7h)"
            Queue2[Task Queue<br/>task_456<br/>remaining: 1h<br/>status: pending]
            Monitor[스케줄러 모니터링<br/>매 10분마다 체크]
        end
        
        subgraph "실행 시점 (T+8h)"
            Queue3[Task Queue<br/>task_456<br/>delay: 0<br/>status: ready]
            Trigger[실행 트리거<br/>AI 서비스 호출]
        end
    end
    
    Queue1 --> Queue2
    Queue2 --> Monitor
    Monitor --> Queue3
    Queue3 --> Trigger
    
    style Queue1 fill:#fff3e0,color:#000
    style Queue2 fill:#e8f5e9,color:#000
    style Queue3 fill:#c8e6c9,color:#000
    style Trigger fill:#ffcdd2,color:#000
```

## 3. JWT 인증 토큰 만료 문제

### 토큰 생명주기와 스케줄링 충돌

```mermaid
timeline
    title JWT Token 생명주기 vs 8시간 지연 실행

    section 등록 시점 (T+0)
        Token 발급 : 유효기간 1시간
        스케줄 등록 : Header Token 사용
        
    section 1시간 후 (T+1h)
        Token 상태 : 만료됨
        스케줄 상태 : 대기 중 (7시간 남음)
        
    section 실행 시점 (T+8h)
        Token 상태 : 8시간 전 만료
        AI 처리 : 토큰 없음 - 실행 불가
```

### 해결 방안: Internal API Token(Key)

#### Internal API Token(Key)란?
- **정의**: 서비스 간 통신을 위해 사전에 발급된 고정 API Key로, 사용자 인증과 독립적으로 작동
- **목적**: 장기 실행 작업에서 사용자 토큰 만료 문제를 우회하여 서비스 간 안전한 통신 보장
- **특징**: 각 서비스가 보유하는 비밀 키로, 만료 시간이 없거나 매우 긴 수명을 가짐

#### 일반 사용자 토큰과의 차이점
- **만료 시간**
  - 사용자 토큰: 1-2시간 (보안상 짧은 만료)
  - Internal API Token: 무기한 또는 매우 긴 만료 (월/년 단위)

- **인증 방식**
  - 사용자 토큰: JWT 기반 동적 토큰
  - Internal API Token: 사전 발급된 고정 API Key

- **사용 목적**
  - 사용자 토큰: 사용자 인증 및 권한 확인
  - Internal API Token: 서비스 간 통신 인증만 담당

#### 장기 실행 작업에서의 활용 방법

1. **스케줄 등록 시점 (T+0)**
   - 사용자 토큰으로 스케줄 등록 요청 인증
   - Main API가 사용자 정보(`user_id`, `task_details`)를 추출하여 저장
   - AI Service가 사용자 컨텍스트와 함께 작업을 큐에 저장
   - 사용자 토큰은 등록 후 더 이상 사용하지 않음

2. **실행 시점 (T+8h)**
   - AI Service가 Internal API Key로 다른 서비스들과 통신
   - Storage Service 호출: `Authorization: Bearer INTERNAL_AI_KEY`
   - Notification Service 호출: `Authorization: Bearer INTERNAL_AI_KEY`
   - 사용자 컨텍스트는 API 파라미터로 전달 (`user_id`, `task_id`)

#### Internal API Key 관리 방식

- **서비스별 API Key 할당**
  - AI Service → Storage: `INTERNAL_AI_TO_STORAGE_KEY`
  - AI Service → Notification: `INTERNAL_AI_TO_NOTI_KEY`
  - Storage Service → Auth: `INTERNAL_STORAGE_TO_AUTH_KEY`

- **키 저장 및 관리**
  - 환경변수 또는 보안 저장소(Vault)에 저장
  - 각 서비스는 자신에게 할당된 키만 보유
  - 정기적인 키 로테이션 (월/분기별)

- **헤더 구성**
  - `Authorization: Bearer INTERNAL_SERVICE_KEY`
  - `X-Service-Name: ai-service` (호출자 식별)
  - `X-User-Context: user_123` (원본 사용자 정보)


## 5. AI 처리 실행 단계 (8시간 후)

### AI 이미지 처리 플로우

```mermaid
sequenceDiagram
    participant Scheduler as Task Scheduler (내장)
    participant AIAPI as AI Service & Task Queue
    participant StorageAPI as Storage Service
    participant S3 as S3 Storage
    participant NotiAPI as Notification Service
    
    Scheduler->>AIAPI: 8시간 후 자동 실행 트리거<br/>task_id: task_456
    AIAPI->>AIAPI: 작업 상태 업데이트<br/>status: "processing"
    
    Note over AIAPI,StorageAPI: Internal API Key 인증 방식
    AIAPI->>StorageAPI: GET /api/v1/storage/download<br/>Authorization: Bearer INTERNAL_AI_TO_STORAGE_KEY<br/>X-Service-Name: ai-service<br/>X-User-Context: user_123<br/>file_id: img_123
    StorageAPI->>StorageAPI: Internal API Key 검증<br/>+ 서비스 권한 확인
    StorageAPI->>S3: 원본 이미지 다운로드
    S3->>StorageAPI: 10MB 이미지 데이터
    StorageAPI->>AIAPI: 이미지 스트림 전달
    
    AIAPI->>AIAPI: GPU 이미지 처리<br/>(5분 소요)
    
    Note over AIAPI,StorageAPI: 처리된 이미지 업로드
    AIAPI->>StorageAPI: POST /api/v1/storage/upload<br/>Authorization: Bearer INTERNAL_AI_TO_STORAGE_KEY<br/>X-Service-Name: ai-service<br/>X-User-Context: user_123<br/>+ 처리된 이미지
    StorageAPI->>StorageAPI: Internal API Key 검증
    StorageAPI->>S3: 결과 이미지 업로드
    StorageAPI->>AIAPI: {"file_id": "result_789", "url": "https://..."}
    
    AIAPI->>AIAPI: 작업 완료 상태 업데이트<br/>status: "completed", result_url
    
    Note over AIAPI,NotiAPI: 사용자 알림 발송
    AIAPI->>NotiAPI: POST /api/v1/notification/send<br/>Authorization: Bearer INTERNAL_AI_TO_NOTI_KEY<br/>X-Service-Name: ai-service<br/>X-User-Context: user_123<br/>{"message": "AI 처리 완료", "result_url": "..."}
    NotiAPI->>NotiAPI: Internal API Key 검증<br/>+ 사용자에게 이메일/푸시 발송
```

