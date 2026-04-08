# risk-service load test

이 폴더는 risk-service의 **부하/증명 테스트** 시나리오와 실행 환경, 결과 박제를 모은다.
[sportsbook/CLAUDE.md](../../CLAUDE.md) "테스트 및 증명 전략"의 Level 2 / Level 3에 해당.

## 목표 숫자

| 지표 | 목표 | 출처 |
|---|---|---|
| `POST /internal/v1/risk/check` p99 | **< 30 ms** at 5 000 동시 사용자 | 시스템 전체에서 가장 빡빡한 latency 요구 |
| `POST /internal/v1/risk/check` p50 | < 5 ms | 본 repo 자체 baseline |
| Kafka `bet.placed` produce throughput | 10 만 events/sec (단일 broker) | sportsbook/CLAUDE.md Level 2 |
| 정합성 invariant | 한도 임박 race → 한도 초과 베팅 모두 거절 | Level 3 |

## 디렉터리 구조

```
load-test/
├── README.md                  본 파일
├── docker-compose.yml         Redis + 단일-노드 Kafka (KRaft mode)
├── scenarios/
│   ├── check_latency.js       k6 — 5 000 VU, 60 s constant
│   └── consumer_throughput.sh kafka-producer-perf-test 래퍼 (Confluent built-in)
└── results/
    ├── BEST.md                최고 성능 박제 (README "성능" 섹션이 여길 가리킴)
    └── <YYYY-MM-DD>/          실행별 raw 결과
        ├── check_latency.json k6 --summary-export
        └── consumer_throughput.txt kafka-producer-perf-test 표준 출력
```

## 사전 준비

```sh
# 1) 의존 인프라 (Redis + Kafka) 띄우기
cd load-test
docker compose up -d

# 2) shared-protocol을 mavenLocal에 박아두기
cd ../../shared-protocol && ./mvnw -q install

# 3) risk-service 빌드
cd ../risk-service && ./mvnw -q -DskipTests package

# 4) risk-service 부팅 — load-test 포트 / 인프라로 override
SERVER_PORT=8083 \
REDIS_HOST=localhost REDIS_PORT=6390 \
KAFKA_BOOTSTRAP=localhost:9094 \
java -jar target/risk-service-0.1.0-SNAPSHOT.jar
```

부팅이 정상이면 `curl localhost:8083/actuator/health` 가 200을 돌려준다.

## 시나리오 실행

### check API 지연 시간

```sh
mkdir -p results/$(date +%Y-%m-%d)
k6 run \
  --summary-export results/$(date +%Y-%m-%d)/check_latency.json \
  scenarios/check_latency.js
```

k6는 threshold 위반 시 비-제로 exit code로 끝나므로 CI에 그대로 끼울 수 있다.
환경 변수 `RISK_BASE_URL` 으로 호스트를 바꿔 운영 stack에도 그대로 돌릴 수 있다.

### consumer throughput

```sh
RECORDS=10000 ./scenarios/consumer_throughput.sh \
  | tee results/$(date +%Y-%m-%d)/consumer_throughput.txt
```

본 스크립트는 Confluent의 `kafka-producer-perf-test` 를 컨테이너 내부에서 호출해
brokerside throughput과 producer p99를 측정한다.  
페이로드는 합성 바이트라 risk-service consumer가 Avro 역직렬화에서 실패하지만,
**부하 측정 자체는 produce 측의 한계를 보는 게 목적**이라 의도된 동작.  
실제 Avro 페이로드로 consumer까지 검증하는 시나리오는 retrospective 단계에서 별도
Java/Kotlin producer 로 추가 예정 (V2 후보).

### 정합성 race

> Level 3 정합성 테스트는 자체 Java 멀티스레드 harness로 `RedisLimitOverrideStore`
> + `SlidingWindowCounter` 위에서 수행한다. `src/test/java/.../counter/SlidingWindowCounterTest#hundredConcurrentRecordsLandExactSum` 가 그 시드.
> "한도 임박 동시 베팅 모두 거절" 시나리오는 추후 별도 commit으로 추가.

## 결과 박제 정책

- 모든 실행 결과를 `results/<YYYY-MM-DD>/` 아래에 raw 파일로 박제 (k6 JSON, perf-test stdout).
- `results/BEST.md` 가 README "성능" 섹션의 단일 출처. 갱신 시 PR으로.
- 결과 commit은 별도: `test(load): record <시나리오> baseline result`.

## 정리

```sh
docker compose down -v   # 컨테이너 + volume 모두 제거
```
