# Docker Compose Health Check 실습

Spring Boot + MySQL 환경에서 Docker Compose의 `healthcheck`와 `depends_on` 조건을 학습하고 검증한 실습입니다.

---

## 🧩 실습 환경

| 항목 | 내용 |
|------|------|
| OS | Ubuntu (VM 서버) |
| Container Runtime | Docker |
| Orchestration | Docker Compose |
| Backend | Spring Boot (Java 17) |
| Database | MySQL 8.0 |
| Build Tool | Maven |
| Health Check | Docker HEALTHCHECK + Spring Boot Actuator |

- 로컬 Windows에서 JAR 빌드 후 SCP로 Ubuntu 서버에 전송
- 서버에서 Docker Compose를 통해 컨테이너 실행

## ⚠️ 문제 인식

일반적인 Docker Compose 환경에서는 컨테이너가 "실행"되었다고 해서 실제 서비스가 준비된 상태를 보장하지 않는다.

예를 들어:

- MySQL 컨테이너가 먼저 실행되더라도 내부 초기화가 끝나기 전일 수 있음
- 이 상태에서 Spring Boot가 먼저 DB에 접속을 시도하면 실패 발생
- 결과적으로 애플리케이션이 비정상 종료되거나 오류 상태로 실행될 수 있음

👉 즉, **"컨테이너 실행" ≠ "서비스 준비 완료"**

이 문제를 해결하기 위해 healthcheck와 depends_on을 활용한 구조를 설계하였다.


## 🧠 핵심 개념

### 1. Health Check

컨테이너 내부 서비스가 정상적으로 동작 가능한 상태인지 판단하는 메커니즘

- 단순 프로세스 실행 여부가 아닌
- 실제 서비스 응답 가능 여부를 기준으로 판단

예:
- MySQL: `mysqladmin ping`
- Spring Boot: `/actuator/health`

---

### 2. depends_on + condition: service_healthy

```yaml
depends_on:
  db:
    condition: service_healthy

## 프로젝트 구조

```
02.compose/
├── Dockerfile
├── docker-compose.yml
└── step04_empAppMaven-0.0.1-SNAPSHOT.jar
```

---

## 주요 파일

### Dockerfile

```dockerfile
FROM eclipse-temurin:17-jdk

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY step04_empAppMaven-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8084

HEALTHCHECK --interval=10s --timeout=5s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8084/emp/deptall || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

| 옵션 | 설명 |
|---|---|
| `--interval=10s` | 10초마다 헬스체크 실행 |
| `--timeout=5s` | 5초 내 응답 없으면 실패 처리 |
| `--start-period=40s` | 초기 40초 동안은 실패해도 retries 카운트 제외 (앱 기동 시간 확보) |
| `--retries=3` | 3번 연속 실패 시 `unhealthy` 상태로 전환 |

---

### docker-compose.yml

```yaml
services:
  db:
    image: mysql:8.0
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - spring-mysql-net
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$$MYSQL_ROOT_PASSWORD || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  app:
    build: .
    container_name: springboot-app
    restart: always
    ports:
      - "8084:8084"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

volumes:
  db_data:

networks:
  spring-mysql-net:
```

**핵심 포인트:** `depends_on`의 `condition: service_healthy` 설정으로 인해 `db` 컨테이너가 `healthy` 상태가 될 때까지 `app` 컨테이너 시작을 블로킹합니다.

---

## 빌드 및 실행

### 1. Maven으로 JAR 빌드 (테스트 제외)

```bash
mvn clean package -DskipTests
```

### 2. Docker Compose 실행

```bash
# 백그라운드 실행
docker compose up -d

# 컨테이너 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f app
```

### 3. 종료 및 볼륨 삭제

```bash
docker compose down -v
```

---

## 헬스체크 동작 검증

### 정상 상태 확인

두 컨테이너 모두 `healthy` 상태임을 `docker inspect`로 확인합니다.

```bash
# DB 헬스 상태 확인
docker inspect --format='{{.State.Health.Status}}' 02compose-db-1

# App 헬스 상태 확인
docker inspect --format='{{.State.Health.Status}}' springboot-app
```

**결과:**

```
healthy
healthy
```

Spring Boot Actuator로 앱 상태도 확인합니다.

```bash
curl http://localhost:8084/emp/actuator/health
```

**결과 (정상):**

```json
{
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "livenessState": { "status": "UP" },
    "readinessState": { "status": "UP" }
  },
  "status": "UP"
}
```

---

### MySQL 중단 시 App 상태 변화

DB 컨테이너를 강제 중지한 뒤 app 컨테이너 상태를 확인합니다.

```bash
docker stop 02compose-db-1
```

```bash
docker inspect --format='{{.State.Health.Status}}' springboot-app
```

**결과:** app 컨테이너 자체는 `healthy` 상태를 유지합니다.

> **이유:** `depends_on condition: service_healthy`는 **최초 기동 순서만 제어**합니다.  
> 일단 app이 정상 시작된 이후에는 MySQL이 죽어도 app 컨테이너의 생명주기에 영향을 주지 않습니다.

하지만 Actuator 헬스 엔드포인트에서는 DB 연결 실패가 즉시 감지됩니다.

```bash
curl http://localhost:8084/emp/actuator/health
```

**결과 (DB 중단 후):**

```json
{
  "components": {
    "db": {
      "status": "DOWN",
      "details": {
        "error": "org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection"
      }
    },
    "readinessState": { "status": "DOWN" }
  },
  "status": "DOWN"
}
```

---

### MySQL 죽은 상태에서 Compose Up 시 App 생성 여부

MySQL이 처음부터 `unhealthy` 상태인 경우의 동작을 검증합니다.

```bash
docker compose down -v
docker compose up -d
```

**결과:**

```
[+] up 3/4
 ✔ Network 02compose_spring-mysql-net   Created
 ✔ Volume 02compose_db_data             Created
 ⠿ Container 02compose-db-1            Waiting    ← MySQL이 healthy 대기 중
 ✔ Container springboot-app            Created    ← app은 Created 상태로 대기
```

> **결론:** `condition: service_healthy` 설정 시 MySQL이 `healthy`가 되기 전까지 **app 컨테이너는 시작되지 않습니다.**  
> MySQL이 끝내 healthy 상태가 되지 못하면 app도 Running 상태로 전환되지 않습니다.

---

## depends_on condition 옵션 비교

| condition | 동작 |
|---|---|
| `service_started` | 의존 컨테이너 프로세스가 시작되면 app 시작 (DB 준비 여부 무관) |
| `service_healthy` | 의존 컨테이너가 `healthy` 상태가 될 때까지 **app 시작 블로킹** |
| `service_completed_successfully` | 배치성 init 컨테이너가 정상 종료된 후 app 시작 |

---

## 핵심 정리

```
depends_on condition: service_healthy
        ↓
  최초 기동 순서 제어 (MySQL healthy 확인 후 app 시작)
        ↓
  app 기동 이후 MySQL 상태와 app 컨테이너 생명주기는 무관
        ↓
  MySQL 장애 시 → 컨테이너는 살아있음, DB 연결만 실패
```

- **기동 시점 보호:** `condition: service_healthy` → MySQL 준비 전 app 시작 방지
- **런타임 장애 감지:** Spring Boot Actuator `/actuator/health` → DB 연결 상태 실시간 확인
- **자동 재시작:** `restart: always` or `restart: on-failure` → app crash 시 재기동 시도
