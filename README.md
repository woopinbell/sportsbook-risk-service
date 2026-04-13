# risk-service

> **Risk service for an online sportsbook.** Synchronously gates bet placement
> on per-user betting limits and rule-based suspicious-pattern detection. It
> sits on the betting-service critical path, so its latency budget is the
> tightest in the system: **p99 < 30 ms**. Part of a 9-service Java 17 +
> Spring Boot 3.2 + Kafka platform — see the orchestration repo for the full
> architecture.

English overview first (for reviewers landing from GitHub), Korean detail
below.

---

## English overview

### What it does

- **Per-user limits** — daily / weekly / monthly stake totals, single-bet
  maximum, and a per-minute selection-count cap, evaluated over a Redis
  sliding window.
- **Operator overrides** — per-user and per-market limit overrides stored in
  Redis hashes, layered over the `application.yml` policy defaults.
- **Rule-based pattern detection** — `rapid-betting`, `sudden-stake-increase`
  and `repeated-same-selection`, each a strategy bean toggled and tuned from
  `application.yml` (no ML in V1).
- **Synchronous decision API** — `POST /internal/v1/risk/check` returns an
  approve/reject verdict with the breached limit or flagged patterns. This is
  the betting-service critical path.
- **Event sourcing** — consumes `bet.placed` from Kafka to keep the counters
  and pattern history fresh, and publishes `risk.limit.violated` /
  `risk.pattern.suspected` back out.

### Architecture

Redis is the **only stateful store** — there is no relational database in V1.
The sliding-window counters live in sorted sets with a companion sum key kept
in lockstep by a Lua script (atomic expiry-cleanup + sum maintenance in one
round-trip); overrides and pattern history live in Redis hashes / sorted sets.
The decision path is read-only against Redis; the write path is driven entirely
by the `bet.placed` Kafka stream (partition key = `userId` for per-user
ordering), so the betting-service is never blocked on risk bookkeeping.

```
betting-service ──HTTP──▶ POST /internal/v1/risk/check ──▶ verdict
       │                          ▲
       └──Kafka: bet.placed──▶ consumer ──▶ Redis counters + history
                                  │
risk-service ──Kafka──▶ risk.limit.violated / risk.pattern.suspected
admin-api ──HTTP──▶ GET/PATCH/DELETE /internal/v1/risk/limits/{userId}
```

### Tech stack

Java 17, Spring Boot 3.2.11, Maven, Spring Data Redis (Lettuce + commons-pool2),
Spring Kafka, Avro (via shared-protocol), Micrometer + OpenTelemetry +
Prometheus, Logback JSON. Tests: JUnit 5 + AssertJ + Mockito + Testcontainers
(Redis via `GenericContainer`, Kafka). Lint/format: Spotless (google-java-format)
+ Checkstyle. Load: k6 + `kafka-producer-perf-test`.

### Build & run

```sh
# shared-protocol must be in mavenLocal first
cd ../shared-protocol && ./mvnw -q install && cd ../risk-service

./mvnw clean verify          # build + unit/integration tests (Testcontainers)
./mvnw spring-boot:run       # needs Redis + Kafka reachable (see orchestration)
```

Load-test run book: [`load-test/README.md`](load-test/README.md).

### Performance (measured 2026-05-29, single host)

| Scenario | Metric | Measured | Target |
|---|---|---|---|
| `check` @ 1 000 RPS | p50 / p95 / p99 | 2.34 / 11.03 / **30.21 ms** | < 5 / < 15 / **< 30 ms** |
| `check` @ 5 000 VU (saturation) | p95 / p99 | 1.46 s / 1.73 s | no SLO |
| `bet.placed` produce (100k × 256 B) | throughput | **173 010 events/s** | 100 000 |
| 100 concurrent counter records | sum accuracy | exact 100 × stake | no race |

p99 landed at 30.21 ms — 0.21 ms over the 30 ms target on a single un-warmed
run. Single source of truth: [`load-test/results/BEST.md`](load-test/results/BEST.md).

Steady-state numbers above are unrelated to the Redis command timeout, which
is a **failure ceiling, not the SLA**: it sits at 1s (with a separate 5s
connect-timeout) so a cold first connection — JVM boot + first GC, before the
pool is primed — is not mistaken for a failed command. See `docs/commits/011.md`.

### Limitations (intentional V1 scope)

- **Market limits are stored but not yet enforced on the check path** — the
  resolver and override store exist, but `RiskCheckService` evaluates user
  limits only (`RiskCheckRequest` carries no `marketId` yet).
- **Open-exposure limit is unimplemented** — it needs a settlement event to
  decrement exposure, which waits on settlement-service.
- **Kafka integration is verified at unit/mock level**, not with a full
  broker → consumer → Redis end-to-end test.
- **No automated cold-start coverage** — the cold-start Redis timeout fix
  (command 1s / connect 5s) is not guarded by a unit test, since a reused
  Testcontainers Redis is already warm; regression protection lives at the
  full-stack e2e level.
- No ML detection (rules only); no schema registry in V1 (plain Avro, single
  shared-protocol version). See `docs/reflection/` for the full list.

---

## 시스템에서의 위치

[sportsbook](../) 9 repo의 Phase 2 leaf 중 하나. `shared-protocol`만 의존하며, `betting-service`가 베팅 접수 critical path에서 본 서비스를 동기 호출한다. 한도 갱신은 `betting-service`가 발행하는 `bet.placed` Kafka 이벤트를 push-consume하여 event sourcing 방식으로 유지한다.

- **의존**: [shared-protocol](../shared-protocol) (value object + RFC 7807 + Avro 이벤트 스키마)
- **의존받음**: [betting-service](../betting-service) (베팅 접수 직전 동기 HTTP check)
- **운영 호출**: [admin-api](../admin-api) (한도 override 변경)

전체 시스템 컨텍스트와 cross-cutting 결정은 [../CLAUDE.md](../CLAUDE.md) 및 [orchestration/docs/architecture/decisions/](../orchestration/docs/architecture/decisions/) 참고.

## 책임 범위

**DO**:
- per-user 한도 — 일/주/월 stake 누적, single bet max stake, 분당 selection 수
- per-market 한도 — market 단위 stake / liability (저장 계층까지 구현, check 경로 미연결 — 아래 제한)
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
│   │   │   ├── pattern/       # 패턴 룰 엔진 (전략 패턴) + Redis history
│   │   │   ├── policy/        # @ConfigurationProperties (yaml binding)
│   │   │   └── event/         # Kafka consumer (bet.placed) + publisher
│   │   └── resources/
│   │       ├── application.yml
│   │       └── scripts/sliding-window.lua
│   └── test/java/com/sportsbook/risk/...
├── load-test/                 # k6 시나리오 + perf-test 래퍼 + 결과 박제
└── docs/
    ├── commits/               # dev 커밋 1개 = 1페이지 + L3/L2 색인
    └── reflection/            # 회고 + 변경 비용 (docs/notes는 Phase 2부터 미생성)
```

## 노출 인터페이스

### HTTP REST (internal)

- `POST /internal/v1/risk/check` — betting-service가 베팅 접수 직전 동기 호출. 응답: `approved` + 위반 한도 / 패턴.
- `GET /internal/v1/risk/limits/{userId}` — 현재 사용자 한도 + 출처(override/policy).
- `PATCH /internal/v1/risk/limits/{userId}` — admin-api 호출용 한도 override.
- `DELETE /internal/v1/risk/limits/{userId}/{limitType}/{currency}` — override 해제.

### Kafka

- consume: `bet.placed` (`com.sportsbook.protocol.event.BetPlacedRequested`)
- publish: `risk.limit.violated` (`RiskLimitViolated`), `risk.pattern.suspected` (`RiskPatternSuspected`)

## 성능

부하/증명 테스트는 `load-test/`에 박제. 최고 성능은 [`load-test/results/BEST.md`](load-test/results/BEST.md), 실행 가이드는 [`load-test/README.md`](load-test/README.md). 2026-05-29 단일 host 측정:

| 시나리오 | metric | 측정값 | 목표 |
|---|---|---|---|
| `POST /internal/v1/risk/check` (1 000 RPS realistic) | p50 / p95 / p99 | **2.34 ms / 11.03 ms / 30.21 ms** | < 5 / < 15 / **< 30 ms** |
| 동일 endpoint, 5 000 VU saturation | p95 / p99 | 1.46 s / 1.73 s | 운영 한계 |
| Kafka `bet.placed` produce (100k records, 256 B) | 처리량 | **173 010 events/s** | 100 000 events/s |
| 정합성 invariant (100 동시 record) | sum 정확도 | 100 × stake 정확 | race 0 |

p99 30.21 ms 는 30 ms 목표에 0.21 ms (0.7 %) 근접. 단일 측정, JIT 워밍업 미적용 상태. 후속 개선 — Lua GC tuning, Lettuce shared vs pool 비교, 측정 전 warm-up phase — 는 [`docs/reflection/`](docs/reflection/)에 박제.

> 위 정상 latency는 Redis command timeout과 무관하다. command timeout은 **목표(SLA)가 아니라 실패 상한**으로, 1s(+ 별도 `connect-timeout: 5s`)로 둬서 콜드 첫 연결(JVM 부팅·첫 GC, 풀이 데워지기 전)이 명령 실패로 오판되지 않게 한다. 통합 e2e에서 드러난 첫 베팅 500을 닫은 결정 — [`docs/commits/011.md`](docs/commits/011.md).

### 관측성 메트릭 (ADR-0007)

`/actuator/prometheus` 가 다음 지표를 노출:

- `risk_check_latency_seconds` (Timer) — check API 응답시간 histogram
- `risk_limit_violations_total{reason}` — 한도 위반별 카운트
- `risk_pattern_flags_total{rule, action}` — 패턴 매치별 카운트

Grafana 대시보드 JSON은 [`orchestration/observability/dashboards/`](../orchestration/observability/dashboards/) 에 박제 예정.

## 제한 사항 (V1 의도적 제외 + 미완)

- **market 한도가 check 경로에 미연결** — `LimitResolver.resolveMarket()`·override store는 구현·테스트됐지만 `RiskCheckService`는 user 한도만 검사한다.
- **open-exposure 한도 미구현** — 정산 이벤트로 노출을 줄여야 정확해 settlement-service에 의존.
- **Kafka 통합이 mock 레벨** — 실제 broker → consumer → Redis e2e 미작성.
- **콜드스타트 자동 검증 부재** — 콜드 first-connection이 200ms command timeout을 넘겨 첫 `/check`가 500이던 통합 버그는 timeout 분리(command 1s / connect 5s)로 닫았지만, 단위 테스트로는 못 막는다(재사용 Testcontainers Redis는 이미 워밍 상태). 회귀 방지는 전 스택 e2e 레벨에 의존.
- ML 기반 사기 탐지 — 룰 기반만.
- 한도 정책 결정 — 운영자가 외부 입력.
- 한도 영속화를 위한 RDB — Redis로 충분 (V2 PostgreSQL `user_limit` 이전 후보).
- Schema Registry 없음 — plain Avro, 단일 shared-protocol 버전 (V2 Apicurio).
- Correlation rule (L3) — [ADR-0012](../orchestration/docs/architecture/decisions/0012-v1-scope-decisions.md).

자세한 회고와 변경 비용은 [`docs/reflection/`](docs/reflection/), 커밋별 학습 색인은 [`docs/commits/README.md`](docs/commits/README.md).
