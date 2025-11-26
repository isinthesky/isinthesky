# MSA 환경에서 Cookie Token과 Header Token 활용법

## 개요

MSA(마이크로서비스) 아키텍처에서는 여러 독립적인 서비스들이 상호 작용하기 때문에, 인증 토큰 관리가 Monolith 환경보다 훨씬 복잡해집니다. 

### Cookie Token vs Header Token 기본 개념

```mermaid
graph LR
    subgraph "Token 전달 방식"
        Cookie[Cookie Token<br/>Set-Cookie: token=xxx<br/>자동 전송]
        Header[Header Token<br/>Authorization: Bearer xxx<br/>수동 설정]
    end
    
    subgraph "특징 비교"
        CookieFeatures[Cookie Token<br/>• 브라우저 자동 관리<br/>• CSRF 위험 존재<br/>• Domain 제한<br/>• HttpOnly 가능]
        HeaderFeatures[Header Token<br/>• 명시적 제어<br/>• CSRF 공격 방어<br/>• CORS 필요<br/>• 저장소 직접 관리]
    end
    
    Cookie --> CookieFeatures
    Header --> HeaderFeatures
    
    style Cookie fill:#e8f5e9,color:#000
    style Header fill:#fff3e0,color:#000
    style CookieFeatures fill:#f3e5f5,color:#000
    style HeaderFeatures fill:#e0f2f1,color:#000
```

### CSRF 보안 차이점 상세 예시

#### Cookie Token의 CSRF 위험 시나리오

```html
<!-- 악의적 사이트 (evil-site.com)에서 -->
<form id="maliciousForm" action="https://mybank.com/api/transfer" method="POST">
  <input type="hidden" name="to" value="attacker-account">
  <input type="hidden" name="amount" value="10000">
</form>

<script>
// 사용자가 mybank.com에 로그인되어 있다면
// Cookie가 자동으로 전송되어 송금이 실행됨
document.getElementById('maliciousForm').submit();
</script>
```

- **문제점**: 브라우저가 mybank.com의 인증 Cookie를 자동으로 전송하므로, 사용자 모르게 송금이 실행됨
- **보안**: Same-Origin-Policy에 의해 악의적 사이트에서는 다른 도메인의 토큰에 접근할 수 없음

### MSA 환경에서의 특별한 고려사항

- **서비스 간 통신**: 각 서비스가 독립적으로 토큰을 검증해야 함
- **도메인 분리**: 서로 다른 서브도메인이나 포트를 사용하는 서비스들
- **클라이언트 다양성**: 웹, 모바일, 서버 투 서버 등 다양한 클라이언트

## Cookie Token 활용법

### 웹 애플리케이션에서의 활용

Cookie Token은 웹 브라우저 환경에서 가장 자연스러운 인증 방식입니다.

```mermaid
sequenceDiagram
    participant Browser as 브라우저
    participant Gateway as Main API Service
    participant Auth as Auth Service
    participant API as Product API Server
    
    Browser->>Gateway: POST /api/v1/auth/login
    Gateway->>Auth: 요청 전달
    Auth->>Auth: 사용자 인증
    Auth->>Gateway: Set-Cookie: auth_token=xxx
    Gateway->>Browser: 로그인 응답 + Cookie 설정
    
    Note over Browser: Cookie 자동 저장
    
    Browser->>Gateway: GET /api/v1/main/products<br/>(Cookie 자동 첨부)
    Gateway->>Auth: 토큰 검증 요청
    Auth->>Gateway: 검증 결과
    Gateway->>API: 인증된 요청 전달
    API->>Gateway: 응답 데이터
    Gateway->>Browser: 최종 응답
```

### 서버간 통신 시 한계점

```mermaid
graph TB
    subgraph "Cookie Token의 한계"
        Problem1[서버 간 통신 불가<br/>Cookie는 브라우저 전용]
        Problem2[도메인 제한<br/>다른 도메인 간 공유 어려움]
    end
    
    subgraph "해결 방안"
        Solution1[Internal API Token<br/>서버 간 통신용 별도 토큰]
        Solution2[Token Extraction<br/>Cookie에서 토큰 추출하여 Header로 변환]
    end
    
    Problem1 --> Solution1
    Problem2 --> Solution2
    
    style Problem1 fill:#ffcdd2,color:#000
    style Problem2 fill:#ffcdd2,color:#000
    style Solution1 fill:#c8e6c9,color:#000
    style Solution2 fill:#c8e6c9,color:#000
```

## Header Token (Bearer Token) 활용법

### API 통신에서의 활용

Header Token은 RESTful API와 서비스 간 통신에서 표준적인 방식입니다.

```mermaid
sequenceDiagram
    participant Client as 클라이언트<br/>(Mobile/Web)
    participant Gateway as Nginx Gateway
    participant Auth as Auth Service
    participant Storage as Storage Service
    
    Client->>Gateway: POST /api/v1/auth/login<br/>{"username", "password"}
    Gateway->>Auth: 요청 전달
    Auth->>Gateway: {"access_token": "eyJ..."}
    Gateway->>Client: JWT 토큰 반환
    
    Note over Client: 토큰 저장 (LocalStorage/SecureStorage)
    
    Client->>Gateway: POST /api/v1/storage/upload<br/>Authorization: Bearer eyJ...
    Gateway->>Gateway: 토큰 검증 or 전달
    Gateway->>Storage: 인증된 요청
    Storage->>Gateway: 파일 업로드 결과
    Gateway->>Client: 최종 응답
```

### 서비스간 통신 최적화

#### Token Relay 방식
- **토큰 전달**: 사용자 토큰을 그대로 다음 서비스로 전달
- **헤더 설정**: `Authorization: Bearer` + `X-Service-Name` 조합
- **사용 사례**: 사용자 권한이 중요한 체인형 서비스 호출

#### Token Exchange 방식  
- **토큰 변환**: 서비스별 전용 토큰으로 변환하여 사용
- **보안 강화**: 각 서비스마다 다른 토큰으로 권한 분리
- **사용 사례**: 높은 보안이 요구되는 민감한 서비스 호출

### Token Relay 패턴

사용자의 원본 토큰을 서비스 체인을 통해 전달하는 패턴입니다.

```mermaid
graph LR
    subgraph "Token Relay Pattern"
        Client[클라이언트<br/>Token: ABC123]
        Gateway[Nginx Gateway]
        MainAPI[Main API<br/>Token: ABC123]
        NotificationAPI[Notification API<br/>Token: ABC123]
        
        Client -->|"Cookie Token: ABC123"| Gateway
        Gateway -->|"Authorization: Bearer ABC123"| MainAPI
        MainAPI -->|"Authorization: Bearer ABC123"| NotificationAPI
    end
    
    subgraph "장단점"
        Pros[장점<br/>• 사용자 권한 유지<br/>• 감사 추적 용이<br/>• 단순한 구현]
        Cons[단점<br/>• 토큰 노출 위험<br/>• 의존성 증가<br/>• 성능 오버헤드]
    end
    
    style Client fill:#e1f5fe,color:#000
    style MainAPI fill:#e8f5e9,color:#000
    style NotificationAPI fill:#f3e5f5,color:#000
    style Pros fill:#c8e6c9,color:#000
    style Cons fill:#ffcdd2,color:#000
```

## 하이브리드 접근법

### 서비스별 라우터 토큰 전략

```mermaid
graph TB
    subgraph "MSA 서비스 구조"
        subgraph "Main API Server"
            MainAPI["/api/v1/main/*<br/>Header Token"]
            MainAdmin["/admin/main/*<br/>Cookie Token"]
        end
        
        subgraph "Auth Server" 
            AuthAPI["/api/v1/auth/*<br/>Header Token"]
            AuthAdmin["/admin/auth/*<br/>Cookie Token"]
        end
        
        subgraph "Storage Server"
            StorageAPI["/api/v1/storage/*<br/>Header Token"]
            StorageAdmin["/admin/storage/*<br/>Cookie Token"]
        end
    end
    
    subgraph "클라이언트"
        WebApp[웹 애플리케이션<br/>API 호출]
        AdminPanel[관리자 대시보드<br/>웹 브라우저]
    end
    
    WebApp -->|Cookie Token| MainAPI
    MainAPI -->|"Header Token<br/>(변환됨)"| AuthAPI
    MainAPI -->|"Header Token<br/>(변환됨)"| StorageAPI
    
    AdminPanel -->|Cookie Token| MainAdmin
    AdminPanel -->|Cookie Token| AuthAdmin
    AdminPanel -->|Cookie Token| StorageAdmin
        
    style MainAPI fill:#e8f5e9,color:#000
    style AuthAPI fill:#e8f5e9,color:#000
    style StorageAPI fill:#e8f5e9,color:#000
    style MainAdmin fill:#f3e5f5,color:#000
    style AuthAdmin fill:#f3e5f5,color:#000
    style StorageAdmin fill:#f3e5f5,color:#000
```

### API 라우터 인증 방식

#### Main API Server (진입점)
- **URL 패턴**: `/api/v1/main/*`
- **웹 앱**: Cookie Token으로 인증
- **외부 파트너**: Header Token으로 인증
- **역할**: 토큰 변환 및 다른 서비스 호출

#### 다른 API Server들
- **URL 패턴**: `/api/v1/{service}/*` (auth, storage 등)
- **인증 방식**: Header Token (Main API에서 변환된 토큰)
- **특징**: 서비스간 통신, 직접 클라이언트 접근 불가

### Admin 라우터: Cookie Token

- **URL 패턴**: `/admin/{service}/*`
- **인증 방식**: Cookie 기반 인증
- **클라이언트**: 관리자 웹 대시보드
- **특징**: 서버 사이드 렌더링, 세션 관리, CSRF 방어 필요

### 서비스간 통신: Internal Token

- **토큰 생성**: 서비스별 고유 Secret으로 JWT 생성
- **스코프 제한**: 필요한 권한만 포함 (`read:notifications`, `write:storage`)
- **서비스 식별**: `X-Service-Name` 헤더로 호출자 식별
- **보안**: 짧은 만료 시간 (1시간) 설정

## 실제 구현 예시

### 각 서비스별 토큰 검증 로직

아래 표는 각 서비스 유형별로 토큰 소스, 검증 방식, 적용 라우트를 요약한 것입니다.

| 구분 | 토큰 소스 | 검증 방식 | 적용 라우트 |
| --- | --- | --- | --- |
| 웹 | Cookie의 `auth_token` | JWT 서명 검증, 만료시간 확인 | 웹 대시보드, 관리자 페이지 |
| API | `Authorization: Bearer` 헤더 | JWT 서명 검증, 스코프 권한 확인 | 모바일 API, 외부 API |
| 내부 | `Authorization` + `X-Service-Name` 헤더 | 서비스별 Secret으로 서명 검증 | 서비스 간 내부 통신 |


## 보안 고려사항

### 서비스별 권한 분리

```mermaid
graph TB
    subgraph "권한 기반 접근 제어 (RBAC)"
        User[사용자<br/>Role: premium_user]
        
        subgraph "서비스별 권한 매핑"
            MainPerms[Main API<br/>• read:products<br/>• write:orders<br/>• read:analytics]
            StoragePerms[Storage Service<br/>• upload:files<br/>• download:own_files<br/>• delete:own_files]
            AIPerms["AI Service<br/>• inference:ai_models<br/>• batch:processing"]
            NotificationPerms[Notification<br/>• send:email<br/>• read:templates]
        end
        
        subgraph "권한 검증"
            TokenScopes[토큰 내 스코프<br/>검증 로직]
            ServiceGuard[서비스별 가드<br/>미들웨어]
        end
    end
    
    User --> MainPerms
    User --> StoragePerms
    User --> AIPerms
    User --> NotificationPerms
    
    MainPerms --> TokenScopes
    StoragePerms --> TokenScopes
    AIPerms --> TokenScopes
    NotificationPerms --> TokenScopes
    
    TokenScopes --> ServiceGuard
    
    style User fill:#e1f5fe,color:#000
    style MainPerms fill:#e8f5e9,color:#000
    style StoragePerms fill:#fff3e0,color:#000
    style AIPerms fill:#f3e5f5,color:#000
    style NotificationPerms fill:#e0f2f1,color:#000
```

### 토큰 스코프 관리

#### 스코프 기반 권한 검증
- **토큰 스코프 확인**: 사용자 토큰에 포함된 권한 범위 검증
- **필수 권한 체크**: API 엔드포인트별 필수 스코프 정의
- **권한 부족 시 처리**: 403 Forbidden 응답 및 필요 권한 안내
