# risk-service 문서 진입점

이 폴더는 risk-service의 **학습·회고 자료**다. 사용자 진입 문서는 repo 루트의
[README.md](../README.md), cross-cutting 결정은 ADR
([orchestration/docs/architecture/decisions/](../../orchestration/docs/architecture/decisions/)).

## 구조

```
docs/
├── README.md       이 파일 (진입점)
├── commits/        dev 커밋 1개 = 1페이지 (000.md ~ 011.md) + 목차·L3/L2 색인 — 답지(정본)
├── practice/       실체 커밋의 문제지(본문 비움) — commits에서 파생
└── reflection/     회고 + 변경 비용 시뮬레이션
```

> `docs/notes/`는 없다. Phase 1(shared-protocol)만 독립 토픽 reference를 가졌고, Phase 2부터
> 중단됐다(2026-05-29 결정). 학습 내용은 각 `commits/NNN.md` 본문과 "기억·설명 Level" 색인으로
> 흡수했다.

## commits/

[commits/README.md](./commits/README.md)가 목차 + 면접 복습용 L3/L2/L1 빠른 참조를 둔다.
12개 dev 커밋을 시간 순으로 1:1 문서화. 각 페이지는 9단 구조.

추천 읽기 순서: 목차의 L3 빠른 참조 → 관심 커밋의 전체 페이지. 핵심 커밋은
[003 counter](./commits/003.md), [005 pattern](./commits/005.md),
[006 api](./commits/006.md), [007 events](./commits/007.md),
[011 콜드스타트 timeout](./commits/011.md).

## practice/

[practice/README.md](./practice/README.md) — `commits/`(답지)에서 본문만 깎아낸 **문제지**.
실체 있는 커밋 7개(001·003·004·005·006·007·009)만 1:1 미러링하고, 마커·문서·thin-config
커밋(000·002·008·010·011)은 답지만 둔다(재유도할 구현이 없어서). 답지가 정본, 문제지는 파생.

## reflection/

- [retrospective.md](./reflection/retrospective.md) — 5단 회고(무엇을 만들었나 / 가설 /
  가설 vs 실제 / 다시 한다면 / 남은 한계). "어디서 시간을 잃었나"가 핵심.
- [change-cost.md](./reflection/change-cost.md) — 6~12개월 변경 시나리오 5종의 깨질 위치 ·
  복구 동선 · 비용, + 의도적으로 미룬 진화 · 재설계 임계점.

## 성능

부하 테스트 결과는 [load-test/results/BEST.md](../load-test/results/BEST.md)가 단일 출처,
실행법은 [load-test/README.md](../load-test/README.md).
