# risk-service 회고 (retrospective)

> Phase 2 leaf 세 번째 repo. dev 12 커밋 + 부하 테스트 baseline까지 끝낸 직후 작성.
> (12번째 dev는 통합 e2e에서 드러난 콜드스타트 Redis timeout fix — 아래 3-(7) 참조.)
> 1차 자료: `git log` + 각 커밋 message body + 코드 diff + 부하 결과(`load-test/results/BEST.md`).

---

## 1. 무엇을 만들었나

베팅 접수 직전에 **betting-service가 동기로 호출하는 한도·패턴 게이트**를 만들었다.
betting-service의 critical path 위에 있으므로 응답 속도가 곧 베팅 전체의 지연이 된다 —
그래서 목표 latency가 시스템에서 가장 빡빡한 **p99 < 30 ms**다.

구현한 것:

- **Redis sliding window 한도 집계** — sorted set + sum key 쌍, Lua script로 만료
  엔트리 정리와 합산을 atomic하게. 일/주/월 stake 누적과 분당 selection 수를 같은 자료구조로.
- **per-user / per-market 한도 override** — Redis hash. 운영자(admin-api)가 쓰고,
  `LimitResolver`가 override → yaml 기본값 순으로 effective limit을 합성.
- **룰 기반 의심 패턴 탐지** — `rapid-betting` / `sudden-stake-increase` /
  `repeated-same-selection` 3종. 전략 패턴(strategy pattern), yaml로 on/off·임계값 조정.
- **동기 check API** — `POST /internal/v1/risk/check`, RFC 7807 에러 응답.
- **운영 한도 API** — `GET/PATCH/DELETE /internal/v1/risk/limits/{userId}`.
- **Kafka event sourcing** — `bet.placed`를 consume해 counter·history를 갱신,
  `risk.limit.violated` / `risk.pattern.suspected`를 publish.
- **부하 baseline** — k6로 check API, `kafka-producer-perf-test`로 produce throughput.

75개 단위·통합 테스트(Testcontainers Redis 포함), spotless·checkstyle 0 위반.

데이터 저장은 **Redis 단독**이다. 이 repo는 V1에서 관계형 DB(relational database)를 쓰지
않는다(CLAUDE.md 결정). sliding window는 sorted set, override는 hash, 패턴 history도
sorted set. 이게 이 repo의 가장 큰 성격이다.

---

## 2. 시작 시점의 가설

세션 시작 가이드(SESSION_STARTUP.md)는 risk-service를 "leaf, 빠르게 끝남"으로 분류했다.
나도 그렇게 가정하고 들어갔다. 구체적 가설은 다음이었다:

1. **Redis sorted set + Lua면 p99 < 30 ms는 충분하다.** Kafka KTable 같은 stateful
   streaming은 V1엔 오버킬(over-engineering)이라고 ADR 단계에서 판단했다.
2. **Kafka push(event sourcing)가 HTTP pull보다 정합성이 명확하다.** betting-service에
   부담을 주지 않고 자체 집계를 유지할 수 있다.
3. **sliding window가 tumbling window보다 사용자 친화적이다.** 자정 직후 한도가 리셋되어
   같은 user가 다시 한도만큼 베팅하는 게이밍(gaming)을 막는다.
4. dev 자체는 leaf답게 빠를 것이고, 마찰은 도메인 로직이 아니라 **인프라 통합**에서 나올 것이다.

가설 4만 맞았고, 마찰의 양은 예상보다 컸다.

---

## 3. 가설 vs 실제 — 어디서 실제로 시간을 잃었나

도메인 로직(한도 합산, 패턴 임계값, override fallback)은 정말 빨리 짰다. C/C++에서 오던
입장에서 Java 17 record + 컴팩트 생성자(compact constructor)로 값 검증을 한 곳에 모으는
패턴은 금방 손에 익었다. 시간은 전부 **그 바깥**에서 샜다.

**(1) 동시성 테스트가 스스로 deadlock했다.**
`SlidingWindowCounterTest`의 "100 동시 record → 정확한 합산" 테스트를 16-thread pool에
100개 task를 submit하는 식으로 짰다. 그런데 ready/start/done `CountDownLatch` 안무에서,
worker 16개가 전부 `start.await()`에 묶이면 나머지 84개 task는 pool에 들어가지도 못해
`ready` latch가 100에 영영 도달하지 못한다. 테스트가 그냥 매달렸다(hang). surefire가 안
끝나서 `jstack`으로 스레드 덤프를 떠서야 `LockSupport.park`에 묶인 main 스레드를 봤다.
고친 건 한 줄 — thread pool 크기를 task 수와 같게(`newFixedThreadPool(threads)`) — 인데,
진단에 시간을 가장 많이 썼다. **테스트가 프로덕션 코드보다 먼저 깨진** 첫 사례였다.

**(2) Avro generated 타입이 내 가정과 달랐다.**
shared-protocol의 Avro plugin이 `timestamp-millis` logical type을 `long`이 아니라
`java.time.Instant`로 매핑(jsr310)하도록 설정돼 있었다. 나는 `setOccurredAt(long)` /
`getRequestedAt()` 가 long을 준다고 가정하고 publisher·consumer를 짰다가 컴파일 에러를
4건 받았다. sources jar를 풀어 generated setter 시그니처를 직접 확인하고서야 고쳤다.
**의존 라이브러리의 generated 코드를 가정하지 말고 먼저 열어봤어야** 했다.

**(3) 런타임에서만 드러난 의존성 누락.**
`application.yml`에 Lettuce pool을 켜놨는데 `commons-pool2`가 클래스패스에 없었다.
단위 테스트는 전부 통과했다 — pool은 부팅 시점에야 초기화되니까. 부하 테스트용으로 jar를
띄우다가 `NoClassDefFoundError: GenericObjectPoolConfig`로 처음 잡혔다. **테스트 그린이
런타임 그린을 보장하지 않는다**는 걸 부하 단계에서 확인했다.

**(4) Spring MVC 마찰 두 건.**
`@WebMvcTest`로 컨트롤러를 띄웠더니 application의 `clock` @Bean과 테스트의 `TestClockConfig`
clock이 이름 충돌(`BeanDefinitionOverrideException`)을 냈다. `standaloneSetup`(MockMvc를
컨트롤러 인스턴스만으로 조립)으로 바꿔 컨텍스트 자체를 안 띄우게 했다. 또 `@PathVariable`이
`-parameters` 컴파일 플래그 없이는 파라미터 이름을 reflection으로 못 찾아 전 한도 API가 400을
뱉었다 — `maven.compiler.parameters=true`로 해결.

**(5) Kafka 부하 환경 셋업.**
KRaft 모드 cluster id가 유효한 UUID가 아니어서 broker가 부팅조차 안 됐다. 그 다음엔
advertised listener를 `localhost:9094` 하나만 둬서, 컨테이너 *내부*의 perf-test admin
client가 metadata를 받고 host 포트로 다시 dial하려다 무한 대기했다. INTERNAL/HOST 두
listener로 분리하고서야 produce가 흘렀다. **도커 네트워킹의 localhost 함정**을 또 밟았다.

**(6) checkstyle MagicNumber.**
룰 기본값(60/30/10/24/5)과 404를 인라인으로 쓸 때마다 빌드가 막혔다. 매번 named constant로
빼는 건 사소하지만 누적되면 흐름이 끊긴다.

**(7) 콜드스타트 Redis timeout — 통합에서만 드러난 버그 #9 (가장 늦게, 가장 멀리서).**
risk 단독 테스트가 전부 그린이고 부하 baseline까지 박은 뒤, 전 스택을 docker-compose로 **콜드로
같이 올린** 통합 e2e에서 **첫 베팅이 HTTP 500**으로 죽었다. risk `/check`가
`RedisCommandTimeoutException`을 던졌고, betting이 그걸 read-timeout으로 받아 500을 냈다.
범인은 `application.yml` 한 줄 — `timeout: 200ms`(command timeout)이었다. 정상 check는 ~30 ms인데,
Lettuce가 **처음 한 번** 연결을 수립하는 그 순간(프로세스 부팅 직후 + JIT 워밍업 전 + 첫 GC +
초기 부하)이 200 ms를 넘겼다. 느린 건 명령이 아니라 **최초 connection init**인데 둘이 같은 200 ms
예산을 쓰고 있었다. 무서운 건 **단독 테스트로는 구조적으로 못 잡는다**는 점이다 —
Testcontainers는 Redis 컨테이너를 재사용(reuse)해 이미 **워밍된** 상태를 주고, JVM도 러너가
데워둬서, "프로세스가 막 떠서 처음 연결하는 순간"이라는 타이밍 자체가 단독 테스트엔 없다. 이건
전 스택 콜드 기동에서만 재현됐다. fix는 command timeout을 1s(=실패 상한, 정상 SLA 아님)로 올리고
`connect-timeout: 5s`를 분리해 **연결 수립 단계와 명령 실행 단계의 시간 예산을 구분**한 것이다.
008(commons-pool2 런타임 누락)에 이어 **"단위 그린 ≠ 런타임 그린"의 두 번째 사례** — 그땐
클래스패스, 이번엔 기동 타이밍이 사각이었다. 자세한 건 [011 커밋 문서](../commits/011.md).

요약하면: **도메인은 가설대로 빨랐고, 인프라·테스트·빌드 경계에서 예상의 몇 배를 썼다.**
가설 1(Redis로 충분)은 맞았다 — p99 30.21 ms로 목표에 0.21 ms까지 갔다. 가설 2(Kafka push)도
구현은 됐지만, 아래 한계에 적듯 **통합 검증까지는 못 갔다**. 그리고 가설 4(마찰은 인프라 통합에서
난다)는 (7)에서 가장 극적으로 맞았다 — 가장 늦게, 시스템 통합 레벨에서, 단독 테스트가 닿지 못하는
지점에서 났다.

---

## 4. 다시 한다면

- **Testcontainers Kafka e2e를 처음부터 짰을 것.** 지금 consumer·publisher는 Mockito
  단위 테스트만 있다. 작업 계획의 "Testcontainers e2e — Kafka consume + Redis 갱신 + check
  응답"은 실제로는 mock 레벨에서 멈췄다. 실제 broker를 띄워 `bet.placed` → counter 증가 →
  `check` 거절까지 한 번에 도는 테스트가 있었다면, Avro Instant 매핑 문제(3-2)와 listener
  함정(3-5)을 부하 단계가 아니라 테스트 단계에서 잡았을 것이다.
- **의존 라이브러리의 generated/런타임 동작을 먼저 확인.** Avro sources jar를 코드 짜기
  전에 열어봤다면, commons-pool2를 application.yml에 pool 켤 때 같이 넣었다면, 두 번의
  재작업이 없었다.
- **부하 측정에 warm-up phase를 처음부터.** p99 30.21 ms는 단일 측정·JIT 워밍업 없이 나온
  수치다. 측정 전 60초 warm-up을 k6 시나리오에 넣었다면 목표 안쪽 수치를 더 정직하게 보였을 것.
- **market 한도를 check API까지 연결하거나, 아니면 명시적으로 scope 밖이라 적었을 것.**
  `LimitResolver.resolveMarket()`을 만들고 테스트까지 했는데 `RiskCheckService`가 그걸 안
  쓴다. 만들다 만 인터페이스가 코드에 남았다(아래 한계 참조).

---

## 5. 남은 한계 (의도적으로 닫지 않은 범위)

ADR-0012 V1 scope 결정 및 이 repo의 의도적 제외와 연결해 솔직히 적는다.

- **market 한도가 check 경로에 연결돼 있지 않다.** `LimitResolver.resolveMarket()`과
  `RedisLimitOverrideStore`의 market namespace는 구현·테스트됐지만, `RiskCheckService.doCheck`는
  user 한도만 검사한다. `RiskCheckRequest`에 marketId 필드 자체가 없다. CLAUDE.md 책임 범위의
  "per-market 한도(마켓별 최대 stake, liability)"는 **저장 계층까지만 만들어졌고 결정 경로엔
  미연결**이다. 이건 의도라기보단 작업 6에서 user path를 먼저 닫고 market을 후속으로 미룬 것.
- **open-exposure 한도 미구현.** yaml에 기본값이 있고 `RiskLimitProperties`에 필드도 있지만,
  `LimitType` enum엔 없고 check에서 검사하지 않는다. open exposure는 "미정산 베팅 합계"라
  정산(settlement) 이벤트를 consume해 베팅이 정산될 때 노출을 줄여야 정확한데, 그 consumer가
  없다. settlement-service가 아직 없으므로 자연스러운 미룸이다.
- **Kafka 통합 검증이 mock 레벨.** consumer가 실제 Avro 바이트를 받아 Redis를 갱신하는 e2e가
  없다. `BetPlacedConsumerTest`는 `AvroCodec.encode`로 만든 바이트를 직접 넣어 consumer 메서드를
  호출할 뿐, 실제 broker→listener 경로는 안 탄다.
- **부하 테스트의 consumer 경로 미검증.** `consumer_throughput.sh`는 합성 바이트(synthetic
  payload)를 produce해 broker throughput만 본다. risk-service consumer는 그 바이트를 Avro
  역직렬화하다 실패한다 — 의도된 동작이지만, "10만 events를 실제로 다 consume"하는 증명은 아니다.
  Avro 호환 producer는 V2 후보로 `load-test/README.md`에 박제했다.
- **p99 0.21 ms 미달.** 목표 30 ms에 30.21 ms. 단일 측정, warm-up 없음. README·BEST.md에
  미달 사실과 개선 동선(Lua GC tuning / Lettuce shared vs pool / warm-up)을 같이 박제했다.
- **Level 3 "한도 임박 동시 베팅 모두 거절" race 미작성.** sliding window counter의 합산
  정합성(100 동시 record → 정확한 합)은 `SlidingWindowCounterTest`로 증명했지만, "한도 직전에
  동시 베팅이 들어오면 초과분이 전부 거절되는가"는 별도 harness가 필요하고 후속으로 미뤘다.
  주의: 현재 check는 read-only peek이라 read와 record 사이에 TOCTOU(time-of-check to
  time-of-use) 틈이 있다 — 두 베팅이 동시에 한도 직전 상태를 읽으면 둘 다 통과할 수 있다.
  이 race를 닫으려면 check+record를 한 Lua script로 합치는 설계가 필요하고, 그건 V1 scope 밖이다.
- **wire schema와 내부 모델 불일치.** Avro `RiskLimitType`은 STAKE_DAILY / OPEN_EXPOSURE /
  SELECTIONS_PER_MINUTE만 있어 STAKE_WEEKLY / MONTHLY / SINGLE_BET_MAX 위반은 publish되지
  않는다(로그·메트릭엔 남음). 공개 카탈로그(wire)는 shared-protocol을 단일 출처로 두고 V2
  Apicurio evolution으로 넓히는 게 맞다고 판단해 그대로 뒀다.
- **콜드스타트 자동 검증 부재.** 통합 버그 #9(첫 `/check` 500, 3-(7))는 `application.yml`
  timeout 분리(command 1s / connect 5s)로 닫았지만, **그걸 막아줄 자동 테스트는 여전히 없다.**
  단독 Testcontainers는 워밍된 Redis라 콜드 첫 연결 타이밍을 재현하지 못한다 — "프로세스가 막
  떠서 처음 연결하는 순간"을 만들려면 전 스택을 콜드로 같이 올리는 시나리오가 필요하고, 그건
  orchestration의 e2e/카오스 레벨(Level 4) 책임이다. 즉 이 fix는 **회귀 방지가 통합 레벨에
  의존**하는 상태로 남아 있다.

이 한계들은 `change-cost.md`에서 변경 비용으로 다시 다룬다.
