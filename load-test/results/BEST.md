# risk-service 부하 테스트 — 최고 성능 박제

본 파일이 README "성능" 섹션의 단일 출처. 각 시나리오는 `results/<YYYY-MM-DD>/` 의
raw 파일을 함께 참조.

## 환경

- 머신: macOS (Apple Silicon), 단일 host
- Redis: `redis:7-alpine`, docker-compose `risk-load-redis`, port 6390
- Kafka: `confluentinc/cp-kafka:7.6.1`, KRaft 단일 노드, port 9094, 자동 토픽 생성
- risk-service: `risk-service-0.1.0-SNAPSHOT.jar`, JDK 21 (release 17), Spring Boot 3.2.11

## Level 2 — `POST /internal/v1/risk/check` latency

### realistic_1000rps (목표 대비 검증 시나리오)

| metric | 값 | 목표 | 결과 |
|---|---|---|---|
| p50 | **2.34 ms** | < 5 ms | ✅ |
| p95 | **11.03 ms** | < 15 ms | ✅ |
| p99 | **30.21 ms** | < 30 ms | ❌ 0.21 ms 미달 (1 회 측정) |
| 에러율 | 0.00 % | < 0.1 % | ✅ |
| 처리량 | 1 000 RPS | 1 000 RPS | ✅ |

> **2026-05-29** 실행, k6 `constant-arrival-rate` 1 000 RPS / 60 s, preAllocatedVUs=200.
> Raw: [`2026-05-29/check_latency.json`](./2026-05-29/check_latency.json)

p99 30.21 ms 는 30 ms 목표에 0.7 % 근접. 단일 측정이며 JIT 워밍업 phase 미적용
상태에서 측정. 후속 작업으로 (a) 측정 전 60 s warm-up, (b) Lua script GC 최적화,
(c) Lettuce shared connection vs pool 비교를 후속 commit 으로 박제 예정.

### saturation_5000vu (overload 측정)

| metric | 값 | 비고 |
|---|---|---|
| p95 | 1.46 s | 5 000 VU constant-vus 가 Tomcat thread pool (max=200) 큐를 가득 채움 |
| p99 | 1.73 s | overload 시점의 backlog 대기 |
| 에러율 | 0.04 % | 일부 connection refuse (Tomcat acceptCount 한계) |
| 처리량 | ~1 970 RPS | thread pool saturation 처리 한계 |

> 30 s constant 5 000 VU. saturation 패턴은 SLA 가 아니며 운영 capacity planning
> 자료로 사용.

## Level 2 — Kafka `bet.placed` produce throughput

| metric | 값 | 목표 | 결과 |
|---|---|---|---|
| records | 100 000 | — | 측정 단위 |
| record size | 256 B | — | synthetic payload |
| 처리량 | **173 010 records/s (42.24 MB/s)** | 100 000 events/s (broker) | ✅ |
| producer latency p99 | 257 ms | — | acks=all, batch.size=64 KB, linger.ms=5 |
| producer latency avg | 202 ms | — | — |

> 도구: `kafka-producer-perf-test`, 단일 노드 broker (KRaft, 호스트 머신).
> Raw: [`2026-05-29/consumer_throughput.txt`](./2026-05-29/consumer_throughput.txt).
> consumer 측 Avro deserialise 는 의도적으로 실패 — producer/broker 한계 측정용
> (load-test/README.md 참조). Avro 호환 producer 는 V2 후속.

## Level 3 — 정합성

- `SlidingWindowCounterTest#hundredConcurrentRecordsLandExactSum`:
  100 동시 record → sum 정확히 100 × stake. ✅
- "한도 임박 동시 베팅 모두 거절" race: 후속 commit (별도 멀티스레드 harness).

## 재현 가이드

[`../README.md`](../README.md) 의 "사전 준비" + "시나리오 실행" 참조.
