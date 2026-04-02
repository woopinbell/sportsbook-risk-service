# risk-service

> **English summary** — Risk service for an online sportsbook microservice
> system. Enforces per-user limits (daily / weekly / monthly stake, open
> exposure, single-bet maximum) and per-market limits (stake, liability) on
> top of a Redis sliding-window sorted-set counter, and runs rule-based
> suspicious pattern detection (rapid betting, sudden stake increase,
> repeated same selection) configured in `application.yml`. Synchronously
> gates bet placement via `POST /internal/v1/risk/check` — this is the
> betting-service critical path, target **p99 < 30 ms** at 5 000 concurrent
> requests. Counters are kept fresh by consuming `bet.placed` from Kafka
> (event sourcing); limit violations and pattern hits are published back as
> `risk.limit.violated` / `risk.pattern.suspected`. **Java 17 + Spring Boot
> 3.2 + Maven + Spring Data Redis (Lettuce) + Spring Kafka.** No relational
> database — Redis is the only stateful store in V1. Part of a 9-service
> system; see the orchestration repo for the full architecture.

**구현 진행 중.**

## 시스템에서의 위치

[sportsbook](../) 9 repo의 Phase 2 leaf 중 하나. `shared-protocol`만 의존하며, `betting-service`가 베팅 접수 critical path에서 본 서비스를 동기 호출한다. 한도 갱신은 `betting-service`가 발행하는 `bet.placed` Kafka 이벤트를 push-consume하여 event sourcing 방식으로 유지한다.

- **의존**: [shared-protocol](../shared-protocol) (value object + RFC 7807 + Avro 이벤트 스키마)
- **의존받음**: [betting-service](../betting-service) (베팅 접수 직전 동기 HTTP check)
- **운영 호출**: [admin-api](../admin-api) (한도 override 변경)

전체 시스템 컨텍스트와 cross-cutting 결정은 [../CLAUDE.md](../CLAUDE.md) 및 [orchestration/docs/architecture/decisions/](../orchestration/docs/architecture/decisions/) 참고.

## 책임 범위

**DO**:
- per-user 한도 — 일/주/월 stake 누적, open exposure 합계, single bet max stake
- per-market 한도 — market 단위 stake / liability
- 의심 패턴 룰 — rapid betting, sudden stake increase, repeated same selection (yaml 설정)
- 한도 변경 운영 API (`PATCH /internal/v1/risk/limits/{userId}`)
- 한도 위반 / 패턴 의심 Kafka 이벤트 발행

**DO NOT**:
- 베팅 접수 자체 ([betting-service](../betting-service)의 책임)
- 잔고 관리 ([wallet-service](../wallet-service)의 책임)
- 한도 정책 결정 — 정책은 운영자가 yaml/Redis hash 입력

자세한 책임 경계와 인터페이스는 본 repo의 `CLAUDE.md` (gitignored — Claude 세션 운영 파일) 와 ADR ([0006](../orchestration/docs/architecture/decisions/0006-messaging-and-saga.md), [0009](../orchestration/docs/architecture/decisions/0009-betting-policy-configuration.md), [0013](../orchestration/docs/architecture/decisions/0013-domain-enums.md)) 참고.

## 핵심 결정 (ADR 박제 + repo 결정)

| 항목 | 결정 | 근거 |
|---|---|---|
| 스택 | Java 17 + Spring Boot 3.2.11 + Maven | [ADR-0015](../orchestration/docs/architecture/decisions/0015-stack-pivot-to-java.md) |
| 한도 자료구조 | Redis sorted set sliding window + Lua atomic | repo 결정 (CLAUDE.md) |
| 베팅 정보 수집 | Kafka `bet.placed` push consume (event sourcing) | repo 결정 — `betting-service` 부담 회피 |
| 시간 윈도우 | sliding (자정 reset 게이밍 차단) | repo 결정 |
| 패턴 룰 표현 | yaml 설정 + 전략 패턴 | repo 결정 — 코드 변경 없이 운영자 조정 |
| 한도 override | application.yml 기본값 → Redis hash override | [ADR-0009](../orchestration/docs/architecture/decisions/0009-betting-policy-configuration.md) (V1 RDB 미사용) |
| 응답 시간 목표 | check API **p99 < 30 ms** | sportsbook/CLAUDE.md 테스트 전략 |

## 빌드 / 실행 / 테스트

```sh
# 빌드 (shared-protocol 0.1.0-SNAPSHOT이 mavenLocal에 설치되어 있어야 함)
./mvnw clean verify

# 부팅 (Redis + Kafka가 localhost에서 listen 중이라고 가정 — orchestration/docker-compose 참조)
./mvnw spring-boot:run

# 단위 테스트만
./mvnw test

# Testcontainers 통합 테스트 포함
./mvnw verify
```

shared-protocol 의존 해결법:

```sh
cd ../shared-protocol && ./mvnw clean install   # mavenLocal에 0.1.0-SNAPSHOT 박제
cd ../risk-service
```

## 구조

```
risk-service/
├── pom.xml
├── README.md                  # 본 파일
├── config/checkstyle/         # shared-protocol과 동일 룰 세트
├── src/
│   ├── main/
│   │   ├── java/com/sportsbook/risk/
│   │   │   ├── RiskServiceApplication.java
│   │   │   ├── api/           # REST controller, DTO, exception handler
│   │   │   ├── counter/       # Redis sliding window counter + Lua
│   │   │   ├── limit/         # per-user / per-market override + fallback
│   │   │   ├── pattern/       # 패턴 룰 엔진 (전략 패턴)
│   │   │   ├── policy/        # @ConfigurationProperties (yaml binding)
│   │   │   └── event/         # Kafka consumer (bet.placed) + producer
│   │   └── resources/
│   │       └── application.yml
│   └── test/java/com/sportsbook/risk/...
├── load-test/                 # k6 시나리오 + 결과 박제
└── docs/
    ├── commits/               # retrospective 단계에서 작성
    ├── notes/                 # retrospective 단계에서 작성
    └── reflection/            # retrospective 단계에서 작성
```

## 노출 인터페이스

### HTTP REST (internal)

- `POST /internal/v1/risk/check` — betting-service가 베팅 접수 직전 동기 호출. 응답: `approved` + 위반 한도 / 패턴.
- `GET /internal/v1/risk/limits/{userId}` — 현재 사용자 한도 + 사용량.
- `PATCH /internal/v1/risk/limits/{userId}` — admin-api 호출용 한도 override.

### Kafka

- consume: `bet.placed` (`com.sportsbook.protocol.event.BetPlacedRequested`)
- publish: `risk.limit.violated` (`RiskLimitViolated`), `risk.pattern.suspected` (`RiskPatternSuspected`)

## 성능

부하/증명 테스트 결과는 핵심 기능 완성 후 `load-test/results/`에 박제 + 이 섹션에 숫자 박제. 목표는 sportsbook/CLAUDE.md "테스트 및 증명 전략":

- check API: 5 000 동시 요청, p99 < 30 ms
- Kafka consume throughput: 10 만 events/sec
- 정합성 invariant: 한도 임박 시점 동시 베팅 race → 한도 초과 모두 거절

## 제한 사항 (V1 의도적 제외)

- ML 기반 사기 탐지 — 룰 기반만
- 한도 정책 결정 — 운영자가 외부 입력
- 한도 조회를 위한 RDB 영속화 — Redis hash로 충분 (V2에 PostgreSQL `user_limit` 테이블 이전 후보)
- Correlation rule (L3) — [ADR-0012](../orchestration/docs/architecture/decisions/0012-v1-scope-decisions.md)
