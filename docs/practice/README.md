# risk-service 문제지 (practice)

`commits/NNN.md`(답지·정본)에서 **본문만 깎아낸** 파생물이다. 시그니처·계약·작업 순서·검증은
그대로 두고, 재유도 대상(함수 본문·알고리즘·핵심 값)을 `// TODO: 책임`으로 비웠다.

## 사용법

1. `practice/NNN.md`를 위에서부터 읽는다 — `## 개요` / `## 작업 순서`가 문제 정의다.
2. 각 `### 논리 단위`의 `**책임:**`(무엇/왜)만 보고 빈 TODO를 직접 채운다. "어떻게"는 안 준다.
3. 막히거나 다 풀면 `## 검증`의 명령을 **실제로 돌려** 확인한다(대부분 `./mvnw test -Dtest=...`).
4. 답을 맞춰볼 때만 `../commits/NNN.md`를 연다. (문제지를 직접 수정하지 말 것 — 답지에서 재파생된다.)

> 각 문제지 최상단 배너: `생성물. 직접 수정 금지. 답: ../commits/NNN.md`.

## 목차 (실체 있는 커밋만)

| # | 문제지 | 주제 | 재유도 핵심 |
|---|---|---|---|
| 001 | [001.md](./001.md) | 정책 바인딩 record 2종 | 컴팩트 생성자 방어(normalised) + enabled-gated 검증 |
| 003 | [003.md](./003.md) | sliding window counter | Lua 4단계(원자적 정리·기록·합산) + counter wrapper |
| 004 | [004.md](./004.md) | override + resolver | Redis hash fail-loud + exhaustive switch fallback |
| 005 | [005.md](./005.md) | 패턴 룰 엔진 | 전략 패턴 fold + 룰 3종(median 등) |
| 006 | [006.md](./006.md) | check API | cheap-to-expensive doCheck + history Redis 구현 |
| 007 | [007.md](./007.md) | Kafka events | Avro codec + event sourcing consumer + wire 매핑 |
| 009 | [009.md](./009.md) | 부하 baseline | k6 2단 시나리오 + KRaft listener 분리 + perf 래퍼 |

## 문제지를 만들지 않은 커밋 (답지만 — `../commits/`)

재유도할 구현이 없어 문제지가 공회전이 되는 커밋들. 답지(`commits/NNN.md`)에는 남는다.

- **000** 프로젝트 골격 — pom/yaml/mvnw/checkstyle/진입점. 빌드 config는 재구현이 아니라 복사다.
- **002** `.claude/` 무시 — `.gitignore` 한 줄(마커).
- **008** commons-pool2 — pom 의존 한 줄 추가. 교훈은 개념(테스트 그린 ≠ 런타임 그린)이지 재유도할 코드가 아니다.
- **010** README 성능 섹션 — 순수 문서.
- **011** Redis 타임아웃 fix — `application.yml` 6줄(값 2개). 설계 통찰은 답지가 설명하지만, 재유도는 "값 두 개 세팅"이라 템플릿 채우기다.
