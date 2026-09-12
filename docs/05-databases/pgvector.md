# pgvector

PostgreSQL 확장입니다. 현재 matrix는 PostgreSQL engine의 HNSW(T01)와 IVFFlat(T03)을 측정합니다.

## 설정과 수명주기

`vector.pgvector.index-type`은 `hnsw` 또는 `ivfflat`입니다.

- HNSW: `m=16`, `ef_construction=128`; 검색 `hnsw.ef_search`
- IVFFlat: `lists=10`; 검색 `ivfflat.probes`

IVFFlat은 `CREATE INDEX` 시점의 데이터로 centroid를 학습합니다. `rebuild()`는 테이블만 만들고, 전체 벡터를 적재한 뒤 `awaitReady()`가 실제 행 수를 확인하고 IVFFlat 인덱스를 생성합니다. 인덱스 생성이 끝나야 검색 파라미터 sweep을 시작하며, 이 생성 시간은 `time_to_index_ready_ms`에 포함됩니다. HNSW는 기존처럼 빈 테이블에 인덱스를 만든 뒤 벡터를 적재합니다.

작은 10k 테이블에서 planner의 exact sequential scan이 끼지 않도록 기본 benchmark는 `enable_seqscan=off`를 검색 세션에 적용합니다.

검색 timer에는 JDBC parameter 설정과 SQL 실행·row mapping이 포함됩니다.

## 필터

metadata는 JSONB이고 `metadata -> ? = ?::jsonb`로 필터해 문자열·숫자·boolean 타입을 보존합니다. 기본 하네스는 metadata용 별도 B-tree/GIN을 만들지 않으므로 `filtered_*` 결과를 반드시 따로 봅니다. 운영 설계를 검증할 때는 pgvector iterative scan과 실제 metadata index 전략을 별도 case로 추가해야 합니다.

## 크기

실제 생성한 HNSW 또는 IVFFlat 인덱스 이름을 사용해 `pg_relation_size`를 기록합니다.

## 최신 fairness-v2 실측 — 2026-09-13 완료

[최신 620개 결과](../07-results/fairness-v2-results-20260913.md) 중 이 DB의 실측입니다. 범위는 **모든 검색 파라미터와 다섯 재구축 회차**의 혼합 지표입니다. 최소 p95와 최대 Recall이 같은 점이라는 뜻은 아닙니다. 합성 10k·1024차원·동시성 10 조건이며 같은 Recall 수준의 설정을 6개 산포도로 비교합니다.

| 구성 | 실제 검색 그리드 | 점 수 | Recall@10 범위 | 전체 p95 ms 범위 |
|---|---|---:|---:|---:|
| T01 / PostgreSQL / hnsw | ef_search: 10, 20, 40, 80, 120, 200, 400, 800, 1000 | 45 | 0.668041–1.000000 | 3.809–57.979 |
| T03 / PostgreSQL / ivfflat | probes: 1, 2, 3, 4, 5, 6, 8, 10 | 40 | 0.827835–1.000000 | 9.422–84.193 |

85개 중 워밍업 미달 7개를 포함해 모두 보존합니다. 요청마다 `set_config`와 벡터 검색을 별도 SQL로 보내는 비용이 포함되므로 DB 엔진 자체의 시간으로 단정하지 않습니다. [최신 보고서](../07-results/fairness-v2-results-20260913.md)에서 5회 산포·CPU·RAM·QPS를 함께 보고, 합의되지 않은 2GiB·30ms로 탈락시키지 않습니다. [2026-09-11 v1 결과](../07-results/sweep-results-20260911.md)는 역사 자료입니다.

## 참고

- [pgvector 공식 문서](https://github.com/pgvector/pgvector)
