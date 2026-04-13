# risk-service 변경 비용 시뮬레이션 (change cost)

> 6~12개월 안에 현실적으로 들어올 만한 변경 요청을 상상하고, 각각이 **어디를 깨뜨리는지**,
> **복구 동선**, **비용**을 표로 정리한다. 목적은 "지금 설계가 어떤 변경에 약한가"를 미리
> 드러내는 것. 핵심 데이터 모델·API·의존·정책 기준으로 식별했다.

## 이 repo의 변경에 민감한 표면

- **데이터 모델**: Redis 키 스키마 — `limit:user:{id}:{type}` (sorted set + `:sum`),
  `limit:override:{user|market}:{id}` (hash), `history:user:{id}:bets` / `:sel:{id}`.
  member 인코딩 `"{betId}|{amount}"`.
- **API**: `POST /internal/v1/risk/check` (betting-service critical path),
  한도 관리 `GET/PATCH/DELETE /internal/v1/risk/limits/{userId}`.
- **의존**: shared-protocol Avro 이벤트(`BetPlacedRequested` / `RiskLimitViolated` /
  `RiskPatternSuspected`), Redis, Kafka.
- **정책**: `application.yml`의 `risk.limits` / `risk.patterns` (`@ConfigurationProperties`).

---

## 변경 시나리오

### 1. open-exposure 한도 구현 (settlement 연동)

미정산 베팅 합계 한도를 실제로 강제하라는 요구. 정산되면 노출이 줄어야 하므로 settlement
이벤트 consume이 필요하다.

| 항목 | 내용 |
|---|---|
| 변경 요청 | "미정산 노출 200만 원 초과 시 신규 베팅 거절, 정산되면 노출 감소" |
| 깨질 위치 | `LimitType` enum (OPEN_EXPOSURE 추가) · `RiskCheckService.doCheck` (검사 추가) · 새 `BetSettledConsumer` (노출 감소) · sliding window가 아니라 **증감 가능한 누적값**이라 자료구조 재검토 필요 |
| 복구 동선 | open-exposure는 시간 윈도우가 아니라 "정산 전까지 살아있는 합". sorted set TTL 모델과 안 맞음 → 별도 hash(`exposure:user:{id}`)에 INCRBY/DECRBY. settlement-service의 `bet.settled` 스키마 합의(hub) 필요 |
| 비용 추정 | **중** (3~4일). settlement-service 의존 + 새 자료구조. settlement-service 완성 전엔 착수 불가 |

### 2. market 한도를 check 경로에 연결

지금 `LimitResolver.resolveMarket()`은 있는데 `RiskCheckService`가 안 쓴다. 마켓별 stake/
liability 한도를 실제 강제하려면.

| 항목 | 내용 |
|---|---|
| 변경 요청 | "특정 마켓(예: 인기 경기 1X2)에 마켓 단위 stake 상한을 걸어라" |
| 깨질 위치 | `RiskCheckRequest` (marketId 필드 없음 → 추가) · `RiskCheckCommand` · `RiskCheckService` (market 검사 분기) · betting-service 호출부(요청 본문에 marketId 추가) · selection별 marketId라 슬립당 여러 마켓 가능 → liability 합산 로직 |
| 복구 동선 | resolveMarket·override store는 이미 있으니 저장 계층은 재사용. request DTO에 selection별 marketId를 받아 마켓별로 그룹핑 후 검사. betting-service와 인터페이스 변경이라 hub 동기 필요 |
| 비용 추정 | **중** (2~3일). 저장 계층 재사용이 비용을 낮추지만 cross-repo 인터페이스 변경이 있음 |

### 3. Schema Registry 도입 (V2 Apicurio)

다중 환경 배포 또는 외부 partner producer가 생기면 plain Avro 바이트로는 호환성 검증이 안 된다.

| 항목 | 내용 |
|---|---|
| 변경 요청 | "producer/consumer를 독립 배포하니 스키마 호환성을 런타임에 검증해야" |
| 깨질 위치 | `AvroCodec` (schema-id prefix 추가) · `BetPlacedConsumer` · `RiskEventPublisher` · `application.yml`(registry URL) · serializer를 Apicurio/Confluent로 교체 |
| 복구 동선 | `AvroCodec`가 모든 직렬화를 한 곳에 모아둔 게 여기서 값을 한다 — 두 메서드(encode/decode)만 registry serializer로 교체하면 consumer/publisher는 거의 안 건드림. ADR-0014가 이미 Apicurio를 V2로 지목 |
| 비용 추정 | **중** (2~3일). AvroCodec 단일 지점 덕분에 표면이 좁음. registry 컨테이너 운영(orchestration) 추가가 실제 부담 |

### 4. 한도 정책을 yaml → DB로 이전 (ADR-0009 V2 트리거)

운영자가 정책을 주 1회 이상 바꾸기 시작하면 재배포 부담 때문에 DB + hot reload로 옮긴다.

| 항목 | 내용 |
|---|---|
| 변경 요청 | "재배포 없이 정책 기본값을 실시간 변경" |
| 깨질 위치 | `RiskLimitProperties` / `RiskPatternProperties` (`@ConfigurationProperties` → DB-backed) · `LimitResolver`(기본값 출처 변경) · 이 repo에 **RDB가 없으므로** PostgreSQL + Flyway 신규 도입 또는 Redis hash로 기본값까지 이전 |
| 복구 동선 | `LimitResolver`가 override → 기본값 fallback을 이미 추상화했으므로, 기본값 출처를 yaml에서 store로 바꾸는 지점이 좁다. 다만 RDB 도입은 이 repo 성격(Redis 단독)을 바꾸는 큰 결정 → ADR 필요 |
| 비용 추정 | **중~대** (4~6일). 캐시 무효화·hot reload 인프라가 진짜 비용. Redis로만 가면 중, RDB 도입하면 대 |

### 5. 새 패턴 룰 추가

운영 중 새 사기 패턴(예: 동일 디바이스 다계정, 배당 급변 직전 집중 베팅)을 막아달라는 요구.

| 항목 | 내용 |
|---|---|
| 변경 요청 | "한 디바이스에서 5분 내 3개 이상 계정이 같은 selection에 베팅하면 flag" |
| 깨질 위치 | 새 `PatternRule` 구현 1개 추가 · `RiskPatternProperties`에 nested record 1개 · `RiskPatternType` Avro enum(shared-protocol)에 심볼 추가 · 필요시 `UserBetHistory`에 새 질의 메서드 |
| 복구 동선 | 전략 패턴 덕분에 `RuleEngine`은 `List<PatternRule>`을 주입받으니 **엔진은 안 건드린다**. 새 빈만 추가하면 자동 등록. 디바이스 차원 데이터는 현재 history에 없어 새 key·새 질의가 필요할 수 있음 |
| 비용 추정 | 데이터가 이미 있으면 **소** (1일), 새 차원(디바이스)이면 **중** (history 모델 확장 포함 2~3일) |

---

## 의도적으로 미룬 진화

- **check+record의 원자적 결합.** 현재 check는 read-only peek이라 read와 다음 record 사이에
  TOCTOU 틈이 있다(retrospective 5장). V1은 betting-service가 베팅 수락 후 `bet.placed`로
  record하는 비동기 흐름이라 이 틈을 용인한다. 진짜 hard limit이 필요하면 check가 직접
  reserve하는 Lua script(peek+조건부 record)로 합쳐야 하는데, 그러면 거절된 베팅의 reserve를
  되돌리는 보상(compensation)이 또 필요해진다 — V1 scope 밖.
- **ML 기반 탐지.** CLAUDE.md가 룰 기반으로 명시 제한. 전략 패턴이라 ML 룰을 `PatternRule`
  구현으로 끼우는 길은 열려 있다.
- **per-market liability(특정 selection 적중 시 지급액) 한도.** market stake 한도(시나리오 2)와
  별개로, "이 selection이 이기면 얼마 물어주나"를 추적하려면 odds·정산 결과가 필요해 settlement
  연동(시나리오 1)과 묶인다.

## 재설계가 합리적인 임계점

- **Redis 단독이 한계에 닿을 때.** 지금은 sorted set + hash로 충분하다. 하지만 (a) open-exposure
  처럼 증감하는 누적값, (b) 정책의 DB 이전, (c) 감사(audit)용 영속 한도 이력 — 이 셋 중 둘 이상이
  동시에 요구되면 PostgreSQL 도입이 Redis를 억지로 늘리는 것보다 싸진다. 그 시점에 ADR로
  "risk-service에 RDB 도입"을 박제하는 게 맞다.
- **check API가 sync HTTP로 못 버틸 때.** p99가 30 ms를 만성적으로 넘기면(현재 30.21 ms로
  경계선), 옵션은 (a) JDK 21 가상 스레드로 Tomcat 풀 확장, (b) check 결과 캐싱, (c) betting-service가
  일부 한도를 자체 캐시. saturation 측정(5000 VU에서 p99 1.73 s)이 보여주듯 thread pool이 1차
  병목이라, 가상 스레드 전환(ADR-0015 재논의)이 가장 먼저 검토할 카드다.
