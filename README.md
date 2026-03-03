# Yamoyo Backend

> 팀 협업 및 실시간 팀장 선정 플랫폼 백엔드 서버

## Tech Stack

| Category | Technologies |
|----------|-------------|
| **Language** | Java 21 |
| **Framework** | Spring Boot 3.5, Spring Security 6, Spring WebSocket (STOMP) |
| **Database** | MySQL 8.0, Redis |
| **Authentication** | OAuth2 (Kakao, Naver, Google), JWT |
| **Infrastructure** | Docker, Flyway, Firebase Admin SDK |
| **Monitoring** | Prometheus, Spring Actuator |
| **Documentation** | Springdoc OpenAPI (Swagger) |

---

## Architecture Highlights

### 1. Interceptor 기반 인증/인가

Spring MVC Interceptor를 활용한 Controller 레벨 인증/인가 체계 구축

- **OnboardingInterceptor**: JWT claims 기반 온보딩 상태 검증 (DB 쿼리 제거로 성능 최적화)
- **HostCheckInterceptor**: `@HostOnly` 커스텀 어노테이션 기반 팀룸 관리자 권한 검증

```java
@HostOnly
@PostMapping("/rooms/{roomId}/start-game")
public void startGame(@PathVariable Long roomId) { ... }
```

### 2. WebSocket 실시간 통신

STOMP 프로토콜 기반 WebSocket 구현

- 커스텀 JWT 인증 환경에서 `convertAndSendToUser()` 대신 명시적 경로 방식 사용
- StompHandler에서 SUBSCRIBE 시점 경로 기반 보안 검증

```
/sub/room/{roomId}/user/{userId}  // 개인 메시지
/sub/room/{roomId}                 // 룸 브로드캐스트
```

### 3. Redis 게임 상태 관리

WebSocket 멀티플레이 게임의 일시적 상태를 Redis로 관리

```
room:{roomId}:game         (Hash)      - 게임 phase, 참가자, 지원자 등
room:{roomId}:connections  (Set)       - 접속 중인 사용자 목록
room:{roomId}:timing       (SortedSet) - 타이밍 게임 순위
```

- TTL 기반 자동 정리: 정상 종료 시 즉시 삭제, 비정상 종료 시 30분 후 만료

### 4. OAuth2 소셜 로그인

- Kakao, Naver, Google 통합 로그인
- 동일 이메일 계정 자동 연동
- JWT Access/Refresh Token + HttpOnly 쿠키 보안

### 5. FCM 푸시 알림

- Spring Event + `@Async` 기반 비동기 알림 파이프라인
- 다중 디바이스 푸시 알림 지원

### 6. Exception Handling

- `YamoyoException` + `ErrorCode` enum 기반 일관된 예외 처리
- `ApiResponse<T>` 제너릭 클래스로 응답 양식 통일

---

## Project Structure

```
src/main/java/com/yamoyo/be/
├── domain/
│   ├── user/          # 사용자, 소셜 계정, 약관 동의
│   ├── teamroom/      # 팀룸 관리
│   ├── leadergame/    # 팀장 선정 게임 (WebSocket + Redis)
│   ├── meeting/       # 미팅 일정
│   ├── notification/  # 알림
│   ├── fcm/           # Firebase Cloud Messaging
│   ├── security/      # 인증/인가
│   └── ...
├── config/            # 설정 클래스
└── exception/         # 예외 처리
```

---

## Getting Started

### Prerequisites

- Java 21
- Docker & Docker Compose
- MySQL 8.0
- Redis

### Run with Docker

```bash
docker-compose up -d
```

### Run Locally

```bash
./gradlew bootRun
```

### API Documentation

서버 실행 후 Swagger UI 접속:
```
http://localhost:8080/swagger-ui.html
```

---

## Game Flow

```
PENDING → VOLUNTEER (10초)
  ├─ 지원자 0명 → 전원 자동 지원
  ├─ 지원자 1명 → 즉시 팀장 확정
  └─ 지원자 2명+ → GAME_SELECT
      → LADDER/ROULETTE: 즉시 결과
      → TIMING: 전원 제출 시 결과
→ RESULT → 전원 퇴장 → Redis 데이터 삭제
```