---
title: "Iceberg CDC 환경에서 COW → MOR 전환기"
date: 2026-04-03 14:00:00 +0900
categories: [TroubleShooting]
tags: [iceberg, cdc, copy-on-write, merge-on-read, spark, dagster]
---

## 문제 상황

Dagster + Spark 클러스터 환경에서 CDC 데이터를 Iceberg 테이블에 micro-batch로 적재하고 있었다. 운영 중 S3 스토리지가 예상보다 훨씬 빠르게 증가했고, 특정 테이블은 원본 DB 대비 수배에 달하는 용량을 차지하고 있었다.

원인은 Iceberg의 기본 Update 전략인 **Copy-on-Write(COW)**가 CDC 워크로드와 맞지 않았기 때문이다.

## Iceberg의 Update 전략: COW vs MOR

Iceberg는 데이터 변경(Update/Delete) 시 두 가지 전략을 제공한다. 테이블 속성의 `write.delete.mode`, `write.update.mode`, `write.merge.mode`로 설정한다.

### Copy-on-Write (COW)

변경이 발생하면 해당 Row를 포함한 **데이터 파일 전체를 새로 작성**한다.

```
UPDATE 1 row in file_A (1000 rows)
→ file_A 전체를 복사하여 file_A_v2 생성
→ file_A는 스냅샷에서 참조 해제
```

- 읽기 시 항상 최신 파일만 읽으면 되므로 **읽기 성능이 빠르다**
- 1건 변경에도 파일 전체를 다시 쓰므로 **쓰기 비용이 크다**
- 읽기 중심, 변경이 드문 분석 테이블에 적합

### Merge-on-Read (MOR)

원본 파일은 유지하고, **Delete File로 무효화 + 새 값은 별도 파일에 Append**한다.

```
UPDATE 1 row in file_A (1000 rows)
→ Delete File 생성 (file_A의 n번째 Row 무효 기록)
→ 새 Row만 별도 파일에 기록
→ 읽기 시 원본 + Delete File + 새 파일을 merge
```

- 변경된 부분만 기록하므로 **쓰기 비용이 낮다**
- 읽기 시 merge 과정이 필요하므로 **읽기 성능에 오버헤드**가 있다
- 쓰기 중심, 빈번한 Update/Delete가 발생하는 워크로드에 적합

## CDC + micro-batch에서 COW가 문제인 이유

CDC는 row 단위 변경을 지속적으로 스트리밍하는 패턴이고, Spark Structured Streaming의 micro-batch는 짧은 주기로 이 변경분을 Iceberg에 반영한다. 이 조합에서 COW는 치명적이다.

### 쓰기 증폭

micro-batch마다 소량의 변경이 들어오는데, COW는 매번 해당 파일 전체를 재작성한다. batch 주기가 짧을수록 동일 파일이 반복적으로 재작성되면서 I/O와 스토리지 비용이 배수로 증가한다.

```
batch 1: id=100 UPDATE → file_A 전체 재작성 → file_A_v2
batch 2: id=200 UPDATE → file_A_v2 전체 재작성 → file_A_v3
batch 3: id=100 UPDATE → file_A_v3 전체 재작성 → file_A_v4
```

수백 MB 파일이 row 몇 건 때문에 매 batch마다 통째로 복사된다.

### 스냅샷과 파일 누적

매 batch가 새 스냅샷을 생성하고, COW는 스냅샷마다 대형 파일을 새로 만든다. expire 전까지 구/신 파일이 공존하면서 저장 공간이 폭증한다. 실제로 이 상태에서 운영한 결과 S3 스토리지가 25TB, 파일 수 13.5M개까지 증가했다.

### Spark 클러스터 리소스 낭비

COW의 파일 재작성은 Spark executor에서 수행된다. micro-batch마다 불필요한 전체 파일 읽기/쓰기가 발생하면서 클러스터 리소스를 소모하고, 다음 batch 처리가 밀리는 악순환이 생긴다.

## MOR 전환

### 설정 변경

```sql
ALTER TABLE catalog.db.table_name SET TBLPROPERTIES (
  'write.delete.mode' = 'merge-on-read',
  'write.update.mode' = 'merge-on-read',
  'write.merge.mode'  = 'merge-on-read'
);
```

### 전환 효과

| 항목 | COW | MOR |
|---|---|---|
| micro-batch 1건 변경 시 | 데이터 파일 전체 재작성 | Delete File + 신규 Row만 기록 |
| batch당 쓰기 I/O | 높음 (파일 크기 비례) | 낮음 (변경분만) |
| 스냅샷당 신규 파일 크기 | 수백 MB (전체 파일) | 수 KB~MB (Delete File + 변경분) |
| 저장 공간 증가 속도 | 빠름 | 느림 |
| 읽기 성능 | 빠름 | merge 오버헤드 존재 |
| Spark executor 부담 | 높음 | 낮음 |

## Delete File의 정합성: Position Delete vs Equality Delete

MOR에서는 Delete File의 종류에 따라 데이터 정합성이 달라진다.

### Position Delete

데이터 파일의 정확한 경로와 행 위치(offset)를 지정한다. "이 파일의 n번째 Row는 삭제됨"이라는 물리적 위치 기반 삭제다. Iceberg v3에서는 Deletion Vector(비트맵) 방식으로 진화했다.

### Equality Delete

컬럼 값 조건으로 삭제한다. `id=123`처럼 조건에 일치하는 모든 행을 삭제한다.

### CDC에서의 핵심 원칙

**동일 Commit 내에서 같은 id에 대해 Update가 발생하면, 이전 Row는 반드시 Position Delete로 지워야 한다.** Equality Delete를 사용하면 새로 추가된 값까지 함께 무효화되어 데이터 정합성이 깨질 수 있다.

이 원칙이 깨지는 대표적인 상황:
- 동일 id의 이벤트가 다른 Kafka Partition으로 분산되어 다른 Worker에서 처리될 때
- Commit timeout으로 Worker의 `insertedRowMap`이 초기화될 때
- Schema Evolution으로 Writer가 재생성되면서 위치 정보가 유실될 때

## MOR의 트레이드오프: Compaction 필수

MOR은 쓰기 비용을 줄이는 대신, Delete File이 누적될수록 읽기 성능이 저하된다. Dagster에서 compaction을 정기적으로 스케줄링하여 Delete File을 데이터 파일에 병합해야 한다.

```sql
-- 데이터 파일 compaction
CALL catalog.system.rewrite_data_files(
  table => 'db.table_name',
  strategy => 'sort',
  sort_order => 'id ASC'
);

-- dangling delete 정리를 포함한 position delete file compaction
CALL catalog.system.rewrite_position_delete_files(
  table => 'db.table_name'
);

-- 스냅샷 만료
CALL catalog.system.expire_snapshots(
  table => 'db.table_name',
  older_than => TIMESTAMP '2026-04-01 00:00:00'
);

-- 고아 파일 정리
CALL catalog.system.remove_orphan_files(
  table => 'db.table_name'
);
```

compaction 후에는 Delete File이 데이터 파일에 반영되어 읽기 성능이 복구된다. CDC 파이프라인에서는 이 maintenance 작업을 Dagster Schedule로 정기 실행해야 한다.

## 정리

- CDC + micro-batch 환경에서 COW는 batch마다 파일 전체를 재작성하므로 **스토리지와 Spark 리소스를 급격히 소모**한다
- MOR로 전환하면 변경분만 기록하여 **쓰기 비용과 저장 공간 증가를 크게 줄일 수 있다**
- 단, MOR은 Delete File 누적으로 읽기 성능이 저하되므로 **주기적 compaction, snapshot expire, orphan file 정리**를 함께 운영해야 한다
- Delete File의 Position Delete / Equality Delete 차이를 이해하고, **CDC 정합성 원칙을 지키는 것**이 중요하다
- Update 전략 선택은 워크로드 특성에 따라 결정해야 하며, Iceberg 기본값(COW)이 항상 최선은 아니다

## 참고

- [Apache Iceberg - Spark Writes (Row-level Operations)](https://iceberg.apache.org/docs/latest/spark-writes/)
- [Apache Iceberg - Spark Procedures (Maintenance)](https://iceberg.apache.org/docs/latest/spark-procedures/)
- [Apache Iceberg - Table Maintenance](https://iceberg.apache.org/docs/latest/maintenance/)
- [토스증권 - Iceberg에서 CDC 적용하기 (1)](https://toss.tech/article/iceberg-cdc-1)
