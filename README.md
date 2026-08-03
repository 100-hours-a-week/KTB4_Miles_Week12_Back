# Bamboo Board Backend

대나무숲 콘셉트의 익명 게시판 백엔드입니다. 회원 인증, 게시글·댓글·좋아요·신고 기능을 제공하며, 집중되는 게시글 조회 요청은 Redis에서 처리한 뒤 MySQL에 주기적으로 반영합니다.

## 기술 구성

- Java 26
- Spring Boot 4
- Spring Web, Security, Validation
- Spring Data JPA
- H2, MySQL 8, Flyway
- Redis, Redisson
- JWT 기반 인증
- JUnit 5, Mockito, Testcontainers, JaCoCo
- Gradle
- Docker, Nginx
- GitHub Actions 기반 Blue/Green 배포

## 조회수 처리

게시글 조회수는 Redis의 원자적 증가 연산으로 처리합니다. 변경된 게시글은 dirty set에 기록하고, 스케줄러가 분산 락을 획득한 뒤 Redis의 조회수 스냅샷을 MySQL의 분리된 조회수 테이블에 반영합니다. 여러 백엔드 인스턴스가 실행되더라도 한 인스턴스만 반영 작업을 수행하며, Redis는 AOF를 사용해 재시작 이후에도 데이터를 복구합니다.

## 로컬 실행

로컬 프로필은 게시글과 사용자 데이터에 H2 인메모리 데이터베이스를 사용합니다. 조회수 기능에는 별도로 실행 중인 Redis가 필요합니다.

```bash
docker run --name bamboo-redis-local -p 6379:6379 -d redis:7.4-alpine redis-server --appendonly yes --appendfsync everysec
./gradlew bootRun
```

기본 서버 주소는 `http://localhost:8080`이며 상태는 `/actuator/health`에서 확인할 수 있습니다.

## 검증

```bash
./gradlew clean check --no-daemon
```

전체 검증에는 단위·통합 테스트, JaCoCo 커버리지 검증, Testcontainers 기반 MySQL·Redis 테스트가 포함됩니다. Testcontainers 테스트를 실행하려면 Docker가 실행 중이어야 합니다.

## 운영 환경

운영 프로필은 MySQL과 Redis를 사용합니다. MySQL 스키마는 Flyway가 관리하며 Redis는 AOF를 활성화해 데이터를 보존합니다. 필요한 환경 변수 예시는 `deploy/.env.example`에서 확인할 수 있습니다.

## 배포

CI에서 전체 검증을 통과하면 컨테이너 이미지를 생성합니다. 배포가 활성화된 환경에서는 GitHub Actions와 AWS SSM을 통해 새 버전을 비활성 색상에 먼저 실행하고, 상태 확인이 성공하면 Nginx 연결을 전환하는 Blue/Green 방식으로 배포합니다.
