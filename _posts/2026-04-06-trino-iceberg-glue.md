---
title: "Trino - Iceberg + Glue Catalog 연동 분석 쿼리 엔진"
date: 2026-04-06 14:00:00 +0900
categories: [Tech Exploration]
tags: [trino, iceberg, glue, s3, eks]
---

## Trino란?

Trino는 대규모 데이터에 대해 분산 SQL 쿼리를 실행하는 엔진이다. 데이터를 자체적으로 저장하지 않고, Connector를 통해 다양한 데이터 소스(Iceberg, Hive, PostgreSQL, S3 등)에 직접 쿼리한다.

핵심 특징:
- **스토리지와 컴퓨팅 분리** — 데이터가 있는 곳에 쿼리를 보낸다
- **다양한 데이터 소스 연합 쿼리** — 서로 다른 소스의 데이터를 하나의 SQL로 JOIN 가능
- **메모리 기반 처리** — 중간 결과를 디스크에 쓰지 않고 메모리에서 파이프라이닝

## 아키텍처

Trino 클러스터는 **Coordinator**와 **Worker**로 구성된다.

```
Client (SQL)
    │
    ▼
Coordinator
├── Parser / Analyzer   ← 쿼리 파싱 및 검증
├── Planner / Optimizer ← 실행 계획 생성 및 최적화
├── Scheduler           ← Worker에 Task 할당
│
├── Worker 1  ← Task 실행, 데이터 처리
├── Worker 2  ← Task 실행, 데이터 처리
└── Worker N  ← Task 실행, 데이터 처리
```

### Coordinator

클러스터의 두뇌다. 클라이언트로부터 SQL을 받아 파싱하고, 실행 계획을 수립하고, Worker에 작업을 분배한다.

- **Parser / Analyzer**: SQL 구문을 파싱하고 테이블/컬럼 존재 여부를 검증한다
- **Planner / Optimizer**: 쿼리를 Stage 단위의 분산 실행 계획으로 변환한다. 조건 푸시다운, 조인 순서 최적화 등을 수행한다
- **Scheduler**: 각 Stage를 Task로 나누어 Worker에 할당하고, 실행 상태를 추적한다

### Worker

실제 데이터를 처리하는 노드다. Connector를 통해 데이터 소스에서 데이터를 가져오고, Worker 간에 중간 결과를 교환한다.

Worker가 시작되면 Coordinator의 Discovery Service에 자신을 등록한다. Coordinator는 이를 통해 사용 가능한 Worker 목록을 관리한다.

### 쿼리 실행 흐름

```
Query → Stage → Task → Split
```

| 단계 | 설명 |
|---|---|
| **Query** | 클라이언트가 제출한 SQL 전체 |
| **Stage** | 분산 실행 계획의 논리적 단위. 여러 Worker에서 병렬 실행 |
| **Task** | Stage를 구성하는 실제 작업 단위. 각 Worker에서 실행 |
| **Split** | Task가 처리하는 데이터 청크. Connector가 테이블을 Split으로 분할 |

Coordinator가 Connector에 Split 목록을 요청하면, 각 Split을 Task에 할당하여 Worker에서 병렬로 처리한다. Worker 간 중간 결과 교환은 REST API를 통해 이루어진다.

## SPI (Service Provider Interface)

Trino가 다양한 데이터 소스를 지원하는 핵심은 SPI다. Connector는 네 가지 SPI를 구현하여 Trino에 데이터 소스를 연결한다.

| SPI | 역할 |
|---|---|
| **Metadata SPI** | 테이블, 컬럼, 타입 정보 제공 |
| **Data Statistics SPI** | 테이블 크기, 행 개수 등 통계 정보 |
| **Data Location SPI** | 데이터의 물리적 위치 (Split 생성) |
| **Data Stream SPI** | 실제 데이터 읽기/쓰기 |

Iceberg Connector는 이 SPI를 구현하여 Glue Catalog에서 메타데이터를, S3에서 Parquet/ORC 데이터를 읽는다.

## 데이터 모델: Connector → Catalog → Schema → Table

```
Trino Cluster
└── Catalog (iceberg)          ← Connector + 설정
    └── Schema (db_name)       ← Glue Database
        └── Table (table_name) ← Iceberg Table (S3)
```

- **Connector**: 데이터 소스 유형 (iceberg, hive, postgresql 등)
- **Catalog**: Connector + 연결 설정의 인스턴스. 하나의 Connector로 여러 Catalog을 만들 수 있다
- **Schema**: Catalog 내의 데이터베이스
- **Table**: Schema 내의 테이블

쿼리 시 `catalog.schema.table` 형태로 참조한다.

```sql
SELECT * FROM iceberg.dip_prod_db.market_data;
```

## Iceberg + Glue Catalog 연동

### Catalog 설정

Trino의 Iceberg Connector에 Glue Catalog을 메타스토어로 연결한다.

```properties
connector.name=iceberg
iceberg.catalog.type=glue
hive.metastore.glue.region=ap-northeast-2
fs.native-s3.enabled=true
s3.region=ap-northeast-2
iceberg.remove-orphan-files.min-retention=1d
iceberg.expire-snapshots.min-retention=1d
```

| 설정 | 역할 |
|---|---|
| `connector.name=iceberg` | Iceberg Connector 사용 |
| `iceberg.catalog.type=glue` | AWS Glue를 메타스토어로 사용 |
| `fs.native-s3.enabled=true` | S3 네이티브 파일시스템 활성화 |
| `iceberg.remove-orphan-files.min-retention` | 고아 파일 삭제 최소 보존 기간 |
| `iceberg.expire-snapshots.min-retention` | 스냅샷 만료 최소 보존 기간 |

### 데이터 흐름

```
Trino Worker
    │
    ├── Metadata 요청 → Glue Catalog (테이블 스키마, 파티션 정보)
    │
    └── Data 요청 → S3 (Parquet/ORC 파일 직접 읽기)
```

Spark가 CDC로 Iceberg 테이블에 데이터를 적재하면, Glue Catalog에 메타데이터가 등록된다. Trino는 같은 Glue Catalog을 참조하므로 별도 동기화 없이 즉시 쿼리할 수 있다.

### EKS 배포 시 IAM 연동

EKS에서 Trino를 운영할 때, Glue와 S3 접근을 위해 IRSA(IAM Roles for Service Accounts)를 사용한다.

```yaml
serviceAccount:
  create: true
  name: trino
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/trino-eks
```

IAM Role에 필요한 권한:

**조회 전용 (SELECT)**
- **Glue**: `GetDatabase`, `GetDatabases`, `GetTable`, `GetTables`, `GetPartition`, `GetPartitions`, `BatchGetPartition`
- **S3**: `GetObject`, `ListBucket`, `GetBucketLocation`

테이블 속성 변경(`ALTER TABLE SET PROPERTIES`)이나 maintenance(`optimize`, `expire_snapshots`, `remove_orphan_files`)를 Trino에서 수행하려면 쓰기 권한이 추가로 필요하다.

**테이블 관리 / Maintenance 수행 시 추가 권한**
- **Glue**: `UpdateTable`, `DeleteTableVersion`, `BatchDeleteTableVersion`
- **S3**: `PutObject`, `DeleteObject`

### Iceberg 테이블 관리

Trino에서 Iceberg 테이블의 속성을 조회하고 변경할 수 있다.

```sql
-- 테이블 속성 확인
SHOW CREATE TABLE iceberg.db.table_name;

-- format version 변경
ALTER TABLE iceberg.db.table_name SET PROPERTIES format_version = 2;

-- 파티셔닝 변경
ALTER TABLE iceberg.db.table_name
  SET PROPERTIES partitioning = ARRAY['year(ts)', 'category'];
```

### Iceberg Maintenance

Trino의 `ALTER TABLE EXECUTE`로 Iceberg 테이블 maintenance를 수행할 수 있다.

```sql
-- 데이터 파일 compaction (소형 파일 병합)
ALTER TABLE iceberg.db.table_name EXECUTE optimize;

-- 스냅샷 만료
ALTER TABLE iceberg.db.table_name EXECUTE expire_snapshots(retention_threshold => '1d');

-- 고아 파일 정리
ALTER TABLE iceberg.db.table_name EXECUTE remove_orphan_files(retention_threshold => '1d');
```

## Spark와의 역할 분담

| 항목 | Spark | Trino |
|---|---|---|
| 역할 | CDC 적재 (Write) | 분석 쿼리 (Read) |
| 처리 방식 | micro-batch | MPP (대규모 병렬 처리) |
| Catalog | Glue (공유) | Glue (공유) |
| 스토리지 | S3 (Write) | S3 (Read) |
| 적합한 작업 | ETL, 대량 적재, compaction | Ad-hoc 쿼리, 대시보드, 데이터 탐색 |

Spark와 Trino가 동일한 Glue Catalog + S3를 공유하므로, Spark가 적재한 데이터를 Trino에서 즉시 조회할 수 있다. 각자의 강점에 맞게 쓰기는 Spark, 읽기는 Trino로 역할을 분담한다.

## 참고

- [Trino Documentation - Concepts](https://trino.io/docs/current/overview/concepts.html)
- [Trino Documentation - Iceberg Connector](https://trino.io/docs/current/connector/iceberg.html)
