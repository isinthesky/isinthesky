# 내가 앞으로 해결해야할 미션 (MSA 환경에서 ACID 보장 Transaction 문제)

## 개요

MSA 환경에서는 여러 독립적인 서비스에 걸쳐 데이터가 분산되어 있기 때문에, 전통적인 ACID 트랜잭션을 보장하기 어렵습니다. AI 이미지 처리 스케줄링 작업에서 발생할 수 있는 분산 트랜잭션 문제들을 분석합니다.

## Monolith vs MSA Transaction 차이

### Monolith 환경의 단순함

```mermaid
graph TB
    subgraph "Monolith Transaction"
        App[Application]
        DB[(Single Database)]
        
        App -->|"BEGIN TRANSACTION"| DB
        App -->|"1.사용자 인증"| DB
        App -->|"2.파일 업로드 기록"| DB
        App -->|"3.AI 작업 스케줄 등록"| DB
        App -->|"4.사용자 크레딧 차감"| DB
        App -->|"COMMIT/ROLLBACK"| DB
    end
    
    style DB fill:#c8e6c9,color:#000
    style App fill:#fff3e0,color:#000
```

**특징:**

- 단일 데이터베이스에서 ACID 보장
- 실패 시 전체 롤백 가능
- 일관성 보장이 간단함

### MSA 환경의 복잡성

```mermaid
graph TB
    subgraph "MSA Distributed Transaction"
        Client[클라이언트]
        
        subgraph "독립적인 서비스들"
            MainAPI[Main API]
            AuthService[Auth Service]
            StorageService[Storage Service]
            AIService[AI Service]
            PaymentService[Payment Service]
        end
        
        subgraph "독립적인 데이터베이스들"
            MainDB[(Main DB)]
            AuthDB[(Auth DB)]
            StorageDB[(Storage DB)]
            AIDB[(AI Task DB)]
            PaymentDB[(Payment DB)]
        end
    end
    
    Client --> MainAPI
    MainAPI --> AuthService
    MainAPI --> StorageService
    MainAPI --> AIService
    MainAPI --> PaymentService
    
    AuthService --> AuthDB
    StorageService --> StorageDB
    AIService --> AIDB
    PaymentService --> PaymentDB
    MainAPI --> MainDB
    
    style MainDB fill:#ffcdd2,color:#000
    style AuthDB fill:#ffcdd2,color:#000
    style StorageDB fill:#ffcdd2,color:#000
    style AIDB fill:#ffcdd2,color:#000
    style PaymentDB fill:#ffcdd2,color:#000
```

**문제점:**
- 각 서비스가 독립적인 데이터베이스 소유
- 서비스 간 네트워크 호출로 인한 실패 가능성
- 부분 실패 시 일관성 보장 어려움

## AI 스케줄링에서 발생하는 Transaction 문제 시나리오

### 시나리오 1: 파일 업로드 실패와 데이터 불일치

```mermaid
sequenceDiagram
    participant User as 사용자
    participant MainAPI as Main API
    participant AuthAPI as Auth Service
    participant StorageAPI as Storage Service
    participant AIAPI as AI Service
    participant PaymentAPI as Payment Service
    
    User->>MainAPI: AI 스케줄 등록 요청
    
    MainAPI->>AuthAPI: 사용자 인증 확인
    AuthAPI->>MainAPI: ✅ 인증 성공
    
    MainAPI->>PaymentAPI: 크레딧 차감 (10 크레딧)
    PaymentAPI->>PaymentAPI: 크레딧 차감 완료
    PaymentAPI->>MainAPI: ✅ 결제 성공
    
    MainAPI->>StorageAPI: 이미지 파일 업로드
    StorageAPI->>StorageAPI: ❌ 업로드 실패 (디스크 풀)
    StorageAPI->>MainAPI: ❌ 업로드 에러
    
    MainAPI->>AIAPI: AI 작업 스케줄 등록
    AIAPI->>MainAPI: ❌ 파일 없음으로 등록 불가
    
    Note over User,PaymentAPI: 문제 상황: 크레딧은 차감됐지만<br/>실제 작업은 등록되지 않음
```

**문제점:**
- Payment Service에서 크레딧은 이미 차감됨
- Storage Service 실패로 파일 업로드 안됨
- AI Service에 작업 등록 불가
- **결과**: 사용자는 크레딧 손실, 작업은 실행 안됨

### 시나리오 2: 네트워크 타임아웃과 중복 처리

```mermaid
sequenceDiagram
    participant MainAPI as Main API
    participant StorageAPI as Storage Service
    participant AIAPI as AI Service
    
    MainAPI->>StorageAPI: 파일 업로드 요청
    StorageAPI->>StorageAPI: 파일 업로드 성공
    StorageAPI-->>MainAPI: ❌ 네트워크 타임아웃<br/>(응답 못받음)
    
    MainAPI->>MainAPI: 타임아웃으로 실패 인식<br/>재시도 로직 실행
    
    MainAPI->>StorageAPI: 파일 업로드 재시도
    StorageAPI->>StorageAPI: ⚠️ 동일 파일 중복 업로드
    StorageAPI->>MainAPI: ✅ 업로드 성공
    
    MainAPI->>AIAPI: AI 작업 스케줄 등록 (첫 번째)
    AIAPI->>MainAPI: ✅ 작업 등록 완료
    
    MainAPI->>AIAPI: AI 작업 스케줄 등록 (재시도)
    AIAPI->>MainAPI: ✅ 작업 중복 등록
    
    Note over MainAPI,AIAPI: 문제 상황: 동일한 작업이<br/>두 번 스케줄됨
```

**문제점:**
- 네트워크 타임아웃으로 성공/실패 판단 어려움
- 재시도 로직으로 인한 중복 처리
- **결과**: 동일 작업 중복 실행, 자원 낭비

### 시나리오 3: AI 처리 중 Storage 서비스 장애

```mermaid
sequenceDiagram
    participant User as 사용자
    participant AIAPI as AI Service
    participant StorageAPI as Storage Service
    participant PaymentAPI as Payment Service
    
    Note over User,PaymentAPI: T+0: 스케줄 등록 (모든 서비스 정상)
    User->>AIAPI: AI 작업 스케줄 등록
    User->>PaymentAPI: 크레딧 10개 차감
    PaymentAPI->>AIAPI: ✅ 결제 완료
    
    Note over User,PaymentAPI: T+8h: AI 처리 시작
    AIAPI->>StorageAPI: 원본 이미지 다운로드
    StorageAPI->>AIAPI: ✅ 이미지 전달
    AIAPI->>AIAPI: GPU 이미지 처리 시작
    
    Note over User,PaymentAPI: T+8h+2m: 처리 중 Storage 장애 발생
    StorageAPI->>StorageAPI: ❌ Storage Service 다운
    AIAPI->>AIAPI: 처리 계속 (이미 다운로드 완료)
    
    Note over User,PaymentAPI: T+8h+7m: 처리 완료, 업로드 시도
    AIAPI->>StorageAPI: 결과 이미지 업로드 시도
    StorageAPI-->>AIAPI: ❌ Service 다운으로 응답 없음
    AIAPI->>AIAPI: 업로드 재시도 (실패)
    
    Note over User,PaymentAPI: T+8h+10m: 최종 상태
    AIAPI->>AIAPI: ❌ 처리 결과 저장 실패
    AIAPI->>User: ❌ 알림 실패 (결과 URL 없음)
    
    Note over User,PaymentAPI: 문제 상황: 크레딧 차감됐지만<br/>사용자는 결과 받지 못함
```

**문제점:**
- AI 처리는 완료됐지만 결과 저장 실패
- 크레딧은 이미 차감된 상태
- 사용자에게 결과 전달 불가
- **결과**: 작업 완료됐지만 사용자는 결과 받지 못함

### 시나리오 4: 데이터 일관성 문제

```mermaid
graph TB
    subgraph "데이터 일관성 문제"
        subgraph "각 서비스의 데이터 상태"
            AuthDB[Auth Service<br/>사용자 상태: 활성]
            PaymentDB["Payment Service<br/>크레딧: 90 (10 차감)"]
            StorageDB[Storage Service<br/>파일: 업로드 완료]
            AIDB[AI Service<br/>작업 상태: 실패]
            MainDB[Main Service<br/>전체 상태: 성공]
        end
        
        subgraph "사용자 관점"
            UserView[사용자가 보는 상태<br/>✅ 결제 완료<br/>❌ 결과 없음<br/>❓ 재시도 가능?]
        end
        
        subgraph "관리자 관점"
            AdminView[관리자가 보는 상태<br/>❓ 어느 데이터가 정확?<br/>❓ 환불 필요?<br/>❓ 재처리 필요?]
        end
    end
    
    AuthDB --> UserView
    PaymentDB --> UserView
    StorageDB --> AdminView
    AIDB --> AdminView
    MainDB --> AdminView
    
    style AuthDB fill:#c8e6c9,color:#000
    style PaymentDB fill:#ffcdd2,color:#000
    style StorageDB fill:#c8e6c9,color:#000
    style AIDB fill:#ffcdd2,color:#000
    style MainDB fill:#fff3e0,color:#000
    style UserView fill:#e1bee7,color:#000
    style AdminView fill:#e1bee7,color:#000
```

**일관성 문제:**
- **Auth Service**: 사용자는 정상 상태
- **Payment Service**: 크레딧은 차감됨
- **Storage Service**: 파일은 정상 업로드
- **AI Service**: 작업 처리 실패로 기록
- **Main Service**: 전체적으로는 성공으로 기록

**결과**: 각 서비스마다 다른 상태를 갖고 있어 전체 시스템의 일관성 부족

## MSA Transaction 문제의 근본 원인

### 1. CAP 정리의 한계
- **Consistency(일관성)**: 모든 노드가 동일한 데이터를 보장
- **Availability(가용성)**: 시스템이 항상 응답 가능
- **Partition Tolerance(분할 내성)**: 네트워크 분할 상황에서도 동작

MSA에서는 세 가지를 동시에 만족할 수 없어, 보통 가용성과 분할 내성을 선택하고 일관성을 포기하게 됩니다.

### 2. 네트워크의 신뢰성 문제
- 서비스 간 네트워크 호출 실패 가능성
- 타임아웃, 지연, 패킷 손실
- 부분 실패 상황에서의 판단 어려움

### 3. 독립적인 데이터베이스
- 각 서비스가 자체 데이터베이스 소유
- 전체 시스템 차원의 트랜잭션 보장 불가
- 데이터 동기화 복잡성

이러한 문제들로 인해 MSA 환경에서는 전통적인 ACID 트랜잭션 대신 **최종 일관성(Eventual Consistency)**을 목표로 하는 다양한 패턴들이 필요합니다.
