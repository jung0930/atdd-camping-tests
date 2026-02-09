# System Analysis Report

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 프로젝트명 | atdd-tests |
| 목적 | 캠핑 키오스크 애플리케이션에 대한 ATDD(인수 테스트 주도 개발) 테스트 프로젝트 |
| Java 버전 | 17 |
| Gradle 버전 | 8.14 |
| 빌드 스크립트 | build.gradle.kts (Kotlin DSL) |

이 프로젝트는 외부 저장소(`atdd-camping-kiosk`)의 키오스크 애플리케이션을 Docker로 기동한 뒤,
Cucumber BDD 기반의 인수 테스트를 실행하는 구조이다.

---

## 2. 디렉토리 구조

```
atdd-camping-tests/
├── build.gradle.kts              # 테스트 프로젝트 빌드 설정
├── settings.gradle               # rootProject.name = 'atdd-tests'
├── gradlew / gradlew.bat         # Gradle Wrapper
├── docs/                         # 미션 문서
│   ├── 미션요구사항.md
│   └── 미션진행_command.txt
├── dockerfiles/                  # Dockerfile 모음
│   ├── Dockerfile                    # 범용 Spring Boot Dockerfile
│   └── Dockerfile-kiosk              # 키오스크 전용 멀티스테이지 빌드
├── infra/                        # 인프라 구성
│   ├── docker-compose.yml            # 키오스크 앱 컨테이너
│   ├── docker-compose-infra.yml      # MySQL 등 인프라 컨테이너
│   ├── db/
│   │   └── init.sql                  # DB 초기 데이터 (상품, 캠핑사이트, 예약 등)
│   └── wiremock/
│       └── mappings/
│           └── payment-approve.json  # 결제 API 목(Mock) 응답
├── repos/                        # 테스트 대상 애플리케이션 (git clone)
│   └── atdd-camping-kiosk/           # 캠핑 키오스크 Spring Boot 앱
└── src/test/                     # 테스트 코드
    ├── java/com/camping/tests/
    │   ├── RunCucumberTest.java       # Cucumber 테스트 러너
    │   └── steps/
    │       └── SampleSteps.java       # 샘플 Step Definition
    └── resources/features/
        └── sample.feature             # 샘플 시나리오 (헬스 체크)
```

---

## 3. 기술 스택

### 3.1 테스트 프로젝트 (atdd-tests)

| 분류 | 기술 | 버전 |
|------|------|------|
| BDD 프레임워크 | Cucumber | 7.14.0 |
| 테스트 러너 | JUnit 5 (Jupiter) | 5.10.0 |
| API 테스트 | RestAssured | 5.3.2 |
| JSON 처리 | Jackson Databind | 2.17.2 |
| DB 연동 | MySQL Connector/J | 8.3.0 |

### 3.2 인프라

| 분류 | 기술 |
|------|------|
| 컨테이너 | Docker Compose |
| 데이터베이스 | MySQL 8.0 |
| JDK 이미지 | Eclipse Temurin 17 |
| API 목 서버 | WireMock (정적 매핑) |

---

## 4. 인프라 구성 분석

### 4.1 Docker Compose 서비스

| 파일 | 서비스 | 이미지 | 포트 | 용도 |
|------|--------|--------|------|------|
| docker-compose.yml | kiosk | 로컬 빌드 (Dockerfile-kiosk) | 18081→8080 | 키오스크 앱 |
| docker-compose-infra.yml | db | mysql:8.0 | 3306→3306 | MySQL DB |

### 4.2 Dockerfile-kiosk (멀티스테이지 빌드)

```
Stage 1 (build)                          Stage 2 (run)
┌──────────────────────────┐            ┌──────────────────────────┐
│ eclipse-temurin:17-jdk   │            │ eclipse-temurin:17-jdk   │
│                          │            │                          │
│ COPY 소스코드            │  JAR 복사  │ COPY --from=build *.jar  │
│ gradlew bootJar -x test ─┼───────────►│ java -jar app.jar       │
└──────────────────────────┘            └──────────────────────────┘
```

### 4.3 DB 초기 데이터 (init.sql)

| 테이블 | 레코드 수 | 설명 |
|--------|-----------|------|
| products | 12 | 렌탈 장비 + 판매 상품 |
| campsites | 35 | A-1~A-20, B-1~B-15 구역 |
| reservations | 17 | 다양한 날짜의 예약 데이터 |
| sales_records | 5 | 판매 기록 |
| rental_records | 6 | 대여 기록 (워크인 포함) |

### 4.4 WireMock 매핑

| 엔드포인트 | 응답 | 용도 |
|-----------|------|------|
| POST `/v1/payments` | `{ paymentKey, orderId, status: "APPROVED" }` | 결제 승인 목 |

---

## 5. 테스트 구조 분석

### 5.1 테스트 실행 흐름

```
./gradlew test
    └── RunCucumberTest (@Suite)
            └── features/*.feature 로드
                    └── SampleSteps.java (Step Definition 매핑)
```

### 5.2 현재 테스트 현황

| 구분 | 상태 |
|------|------|
| Feature 파일 | 1개 (sample.feature) |
| Step Definition | 1개 (SampleSteps.java) |
| 시나리오 | 헬스 체크 (GET localhost:8080 → 성공 확인) |

현재는 샘플 수준의 테스트만 존재하며, 실제 API 호출 검증이나 DB 상태 확인 등의 인수 테스트는 아직 작성되지 않은 상태이다.

---

## 6. Gradle Task

| Task | 그룹 | 명령어 | 설명 |
|------|------|--------|------|
| `composeUp` | infra | `./gradlew composeUp` | 키오스크 컨테이너 빌드 및 기동 |
| `composeDown` | infra | `./gradlew composeDown` | 키오스크 컨테이너 종료 및 볼륨 삭제 |
| `test` | verification | `./gradlew test` | Cucumber 인수 테스트 실행 |

---

## 7. 외부 의존 관계

```
                    ┌─────────────────────┐
                    │   atdd-tests        │
                    │   (테스트 프로젝트)    │
                    └────────┬────────────┘
                             │ Cucumber + RestAssured
                             ▼
                    ┌─────────────────────┐
 ┌─────────────────►│   kiosk :18081      │◄─────────────────┐
 │  상품 조회/판매확정 │   (Docker 컨테이너)  │ 결제 생성/승인/환불 │
 │                  └─────────────────────┘                  │
 │                                                           │
 ▼                                                           ▼
┌──────────────┐                                 ┌──────────────────┐
│ Admin 서비스  │                                 │ Payment 서비스    │
│ (미구성)      │                                 │ (WireMock :9090) │
└──────────────┘                                 └──────────────────┘
         │
         ▼
┌──────────────┐
│ MySQL :3306  │
│ (atdd-infra) │
└──────────────┘
```

---

## 8. Docker Compose 실행 가이드

### 8.1 사전 준비

- Docker Desktop이 설치되어 있고 실행 중이어야 한다.
- `repos/atdd-camping-kiosk/` 디렉토리에 키오스크 소스코드가 clone 되어 있어야 한다.

### 8.2 인프라 (MySQL) 실행/종료

```bash
# MySQL 컨테이너 기동
docker compose -f infra/docker-compose-infra.yml up -d

# MySQL 컨테이너 종료 및 볼륨 삭제
docker compose -f infra/docker-compose-infra.yml down -v
```

| 항목 | 값 |
|------|-----|
| 컨테이너명 | atdd-db |
| 포트 | 3306 |
| DB명 | atdd |
| Root 비밀번호 | secret |

### 8.3 키오스크 앱 실행/종료

```bash
# Gradle Task 사용 (권장)
./gradlew composeUp      # 빌드 + 컨테이너 기동
./gradlew composeDown     # 컨테이너 종료 + 볼륨 삭제

# docker compose 직접 사용
docker compose -f infra/docker-compose.yml up -d --build   # 빌드 + 기동
docker compose -f infra/docker-compose.yml down -v          # 종료 + 볼륨 삭제
```

| 항목 | 값 |
|------|-----|
| 포트 | 18081 (호스트) → 8080 (컨테이너) |
| 프로필 | local |
| 헬스 체크 | `http://localhost:18081/health` |

### 8.4 전체 실행 순서 (인프라 + 앱 + 테스트)

```bash
# 1. 인프라 기동
docker compose -f infra/docker-compose-infra.yml up -d

# 2. 키오스크 앱 기동
./gradlew composeUp

# 3. 앱 기동 확인 (헬스 체크)
curl http://localhost:18081/health

# 4. 인수 테스트 실행
./gradlew test

# 5. 정리 (역순)
./gradlew composeDown
docker compose -f infra/docker-compose-infra.yml down -v
```

### 8.5 로그 확인 / 트러블슈팅

```bash
# 키오스크 앱 로그 확인
docker compose -f infra/docker-compose.yml logs -f kiosk

# 인프라 로그 확인
docker compose -f infra/docker-compose-infra.yml logs -f db

# 컨테이너 상태 확인
docker ps

# 키오스크 앱 강제 재빌드 (캐시 무시)
docker compose -f infra/docker-compose.yml build --no-cache
docker compose -f infra/docker-compose.yml up -d
```

---

## 9. 식별된 특이사항

1. **WireMock 미기동**: `payment-approve.json` 매핑 파일은 존재하지만, WireMock 컨테이너가 docker-compose에 정의되어 있지 않다.
2. **네트워크 분리**: `docker-compose.yml`(kiosk)과 `docker-compose-infra.yml`(db)이 별도 파일로 분리되어 있으며, 동일 네트워크에 대한 명시적 연결이 없다.
3. **테스트 미구현**: 현재 샘플 시나리오만 존재하며, 실제 비즈니스 시나리오(상품 조회, 결제, 환불 등)에 대한 인수 테스트가 작성되지 않았다.
