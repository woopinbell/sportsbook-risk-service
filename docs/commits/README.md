# risk-service 커밋 문서 목차

dev 커밋 1개 = 문서 1페이지(`NNN.md`). retrospective 커밋(reflection / commits / readme
finalize)은 문서화 대상이 아니다. 각 페이지는 9단 구조(제목 / 개요 / 작업 순서 / 작업 내역 /
결과 / 요약 / 다음 작업 / 핵심 확인 / 기억·설명 Level).

> 실체 있는 커밋(001·003·004·005·006·007·009)은 본문을 비운 **문제지**가
> [`../practice/`](../practice/)에 함께 있다. 이 폴더(답지)가 정본, 문제지는 파생물이다.

## 목차

| # | 문서 | 커밋 | 주제 | 유형 |
|---|---|---|---|---|
| 000 | [000.md](./000.md) | `23c6388` | 프로젝트 골격 (pom / yaml / 진입점) | chore |
| 001 | [001.md](./001.md) | `360ca13` | 정책 바인딩 (`@ConfigurationProperties` record 2종) | feat |
| 002 | [002.md](./002.md) | `ff6bbe3` | `.claude/` 무시 (housekeeping) | chore |
| 003 | [003.md](./003.md) | `75a3025` | Redis sliding window counter (sorted set + Lua) | feat |
| 004 | [004.md](./004.md) | `50505b7` | 한도 override + 정책 fallback resolver | feat |
| 005 | [005.md](./005.md) | `2ba9031` | 패턴 룰 엔진 (전략 패턴) | feat |
| 006 | [006.md](./006.md) | `c3bf24f` | 동기 check API + 한도 관리 엔드포인트 | feat |
| 007 | [007.md](./007.md) | `30cbcea` | Kafka event sourcing + 위험 이벤트 publish | feat |
| 008 | [008.md](./008.md) | `4427fa6` | commons-pool2 (Lettuce pool) | build |
| 009 | [009.md](./009.md) | `410af5a` | 부하/증명 baseline (k6 + perf-test) | test |
| 010 | [010.md](./010.md) | `49d4f16` | README 성능 섹션 박제 | docs |
| 011 | [011.md](./011.md) | `d591b89` | Redis 타임아웃 확대 (콜드스타트 connection init) | fix |
| 012 | [012.md](./012.md) | `1da19e1` | bet-placed 구독 토픽 정렬 (발행측 `.v1` 드리프트) | fix |

작성 일자: dev는 2026-05-28, 부하·README는 2026-05-29, 콜드스타트 fix(011)는 통합 e2e에서
발견돼 2026-05-31. 문서 작성은 retrospective 단계(2026-05-29, 011은 2026-05-31 추가).

**phase 경계**: 000~011 = phase 1(최초 구현 window + retrospective 메타). **012부터 phase 2**
(후속 fix 윈도우, 2026-06-11 시작, 시작 커밋 `1da19e1`) — 경계 규정은 commit-policy.md
§날짜·배치(phase 단위 경계 분리).

---

## L3 빠른 참조 (면접 직전 5분 — 외워서 설명)

가장 깊은 "왜"들. 면접에서 직접 질문받을 가능성이 높은 순.

- **sliding window counter의 sum 불변식** ([003](./003.md)) — sorted set + sum key가 어긋나지
  않는 이유: Lua script가 만료 멤버의 DECRBY와 ZREMRANGEBYSCORE를 한 atomic 단계로 묶는다.
  Redis 단일 스레드라 스크립트 실행 중 끼어듦 없음.
- **check의 TOCTOU 틈** ([003](./003.md), retrospective 5장) — read-only peek이라 두 동시
  베팅이 한도 직전 상태를 같이 읽으면 둘 다 통과 가능. V1이 용인하는 한계. 닫으려면 check+record를
  한 Lua로 합치고 보상(compensation)을 추가해야 함.
- **룰 엔진이 BLOCK에 short-circuit 안 하는 이유** ([005](./005.md)) — BLOCK 의미가 호출자마다
  다름: check는 거절(critical path), `bet.placed` consumer는 이미 접수된 베팅의 이벤트 severity만.
  엔진을 순수 fold로 둬 정책이 새지 않게.
- **wire 스키마 ↔ 내부 모델 불일치를 의도적으로 둔 이유** ([007](./007.md)) — Avro
  `RiskLimitType`은 일부 타입만. STAKE_WEEKLY/MONTHLY 위반은 publish 안 함(로그/메트릭엔 남음).
  공개 카탈로그는 shared-protocol 단일 출처 + V2 Apicurio evolution.
- **sliding vs tumbling window** ([003](./003.md)) — tumbling은 자정 직후 한도 리셋 게이밍 가능.
  sliding은 "정확히 지난 24시간".
- **event sourcing(Kafka push) vs HTTP pull** ([007](./007.md)) — push가 정합성 명확,
  betting-service에 부담 안 줌. partition key=userId로 user 스트림 순서 보장.
- **부하: 왜 5000 VU가 아니라 1000 RPS** ([009](./009.md)) — constant-vus 5000은 saturation,
  constant-arrival-rate 1000 RPS가 서비스 작업을 반영하는 realistic load.
- **command timeout vs connect timeout 분리** ([011](./011.md)) — 콜드스타트 connection
  init이 200ms command timeout을 초과해 첫 `/check`가 false `RedisCommandTimeoutException` →
  betting 500(통합 버그 #9). 연결 수립과 명령 실행은 시간 특성이 달라 하나의 예산으로 묶으면 안
  됨. command 1s(실패 상한, SLA 아님) + connect 5s 분리. 정상 latency 불변. 단독 테스트는
  워밍된 Testcontainers Redis라 콜드 첫 연결 타이밍을 구조적으로 미검출 — 전 스택 콜드에서만 재현.

## L2 빠른 참조 (문서 보며 설명)

- record 컴팩트 생성자에서 방어적 복사 + 기본값 + 검증 ([001](./001.md)).
- 패턴 임계값 검증을 `enabled`일 때만 — runbook 토글 친화 ([001](./001.md)).
- `ApplicationContextRunner`가 `@SpringBootTest`보다 ~50배 싼 이유 ([001](./001.md)).
- override hash field에 type+currency 인코딩 → `DEL` 한 번으로 user 전체 해제 ([004](./004.md)).
- exhaustive switch로 새 한도 추가 시 컴파일 강제 ([004](./004.md)).
- median > mean (이상치 견고), 신규 user·zero-median 가드 ([005](./005.md)).
- cheap-to-expensive 검사 순서로 Redis 왕복 최소화 ([006](./006.md)).
- `-parameters` 플래그 함정 (PathVariable 400) + standaloneSetup vs `@WebMvcTest` ([006](./006.md)).
- AvroCodec 단일 지점이 V2 registry 전환 표면을 좁힘 ([007](./007.md)).
- 빈 selection 슬립은 selection counter skip (sum 불변식 보호) ([007](./007.md)).
- 도커 localhost 함정 + INTERNAL/HOST listener 분리 ([009](./009.md)).
- 단위 테스트 그린 ≠ 런타임 그린 (commons-pool2) ([008](./008.md)).
- Lettuce 연결의 lazy 초기화 — 풀을 켜도 첫 요청에서야 연결이 맺힌다 ([011](./011.md)).
- 콜드 vs 워밍 기동 — 워밍된 Testcontainers Redis는 콜드 첫 연결 타이밍이 없다 ([011](./011.md)).
- 구독·발행 토픽 계약은 발행측 상수가 단일 출처 — 폴백 기본값·테스트 픽스처까지 함께 정렬해야
  fix가 끝난다 ([012](./012.md)).

## L1 빠른 참조 (읽으면 됨)

- 디렉터리 골격, README 영문 박스 ([000](./000.md)).
- 정책 키 목록(stake 일/주/월, single-bet, open-exposure, selections, 패턴 3종) ([001](./001.md)).
- `.claude/` git 제외 ([002](./002.md)).
- Redis 키 두 개(sorted set + :sum), TTL=2*window ([003](./003.md)).
- 엔드포인트 4개, 메트릭 3종 ([006](./006.md)).
