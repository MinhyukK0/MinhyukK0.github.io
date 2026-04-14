---
title: "Log-based CDC: RDS에서 Iceberg까지"
date: 2026-04-14 14:00:00 +0900
categories: [Data Engineering]
tags: [cdc, dms, msk, kafka, dagster, spark, iceberg]
---

## Log-based CDC란?

CDC(Change Data Capture)는 소스 데이터베이스의 변경 사항을 감지하고 다운스트림 시스템에 전파하는 패턴이다. CDC를 구현하는 방식은 여러 가지가 있는데, 그 중 Log-based CDC는 데이터베이스의 트랜잭션 로그를 직접 읽어 변경 사항을 캡처한다.

### 왜 Log-based인가?

데이터베이스는 데이터 변경 시 트랜잭션 로그를 먼저 기록한다. PostgreSQL의 WAL(Write-Ahead Log), MySQL의 binlog가 대표적이다. Log-based CDC는 이 로그를 소비하는 방식이라 소스 DB에 추가 쿼리 부하를 주지 않는다.

### 다른 방식과의 비교

| 방식 | 원리 | 장점 | 단점 |
|------|------|------|------|
| **Polling (Query-based)** | 주기적으로 `updated_at` 등을 기준으로 변경분 조회 | 구현이 단순 | 소스 DB 부하, delete 감지 불가, 폴링 간격만큼 지연 |
| **Kafka Connect (Source/Sink Connector)** | Debezium 등 커넥터가 DB 로그를 읽어 Kafka로 전송 | 생태계가 풍부, 스키마 레지스트리 연동 용이 | 커넥터 관리 오버헤드, 커넥터 장애 시 복구 복잡 |
| **Log-based (DMS)** | 관리형 서비스가 DB 로그를 읽어 스트리밍 | 인프라 관리 최소화, full load + CDC 통합 | 서비스 종속, 세밀한 제어 어려움 |

이전에 Kafka Connect의 Source/Sink Connector를 사용한 적이 있다. 당시 소스 DB 스키마가 변경되었을 때 커넥터가 이를 매끄럽게 반영하지 못해 고생했던 경험이 있다. 물론 Schema Registry 설정이나 커넥터 옵션을 더 잘 활용했다면 달랐을 수 있지만, 적어도 내 경험에서는 스키마 변경에 대한 대응이 번거로웠다.

이런 경험도 있어서 이번에는 DMS를 선택했다. 관리형 서비스라 운영 부담이 줄었지만, 세밀한 설정이 어렵고 스키마 변경 문제가 완전히 사라지는 것은 아니었다. 결국 다운스트림 코드에서 직접 처리하게 되었는데, 이 부분은 뒤에서 다룬다.

---

## 아키텍처 Overview

전체 파이프라인은 다음과 같다.

```
RDS (WAL/binlog) → AWS DMS (CDC) → MSK Serverless (Kafka) → Dagster + Spark (micro-batch) → Apache Iceberg
```

각 컴포넌트의 상세 역할은 아래에서 다룬다.

---

## 각 컴포넌트별 역할

### AWS DMS

DMS는 소스 DB의 트랜잭션 로그를 읽어 타겟으로 스트리밍하는 관리형 서비스다. 태스크 하나로 full load와 CDC를 모두 처리할 수 있다.

- **Full Load**: 최초 실행 시 기존 데이터를 전체 적재한다. 이때 `metadata.operation`은 `load`로 표시된다.
- **CDC**: full load 이후 변경분만 캡처한다. 각 메시지의 `metadata.operation` 필드로 변경 유형을 구분한다.

DMS가 MSK로 보내는 메시지는 JSON 형태이며, 크게 Data Record와 Control Record 두 가지가 있다.

**Data Record** — 실제 행 데이터 변경:

```json
{
  "data": {
    "id": 100000161,
    "fname": "val61s",
    "lname": "val61s"
  },
  "metadata": {
    "timestamp": "2019-10-31T22:53:59.721201Z",
    "record-type": "data",
    "operation": "insert",
    "partition-key-type": "primary-key",
    "schema-name": "public",
    "table-name": "some_table",
    "transaction-id": 9324410911751,
    "commit-timestamp": "2019-10-31T22:53:55.000000Z"
  }
}
```

`metadata.operation` 값은 다음 중 하나다:
- `load`: full load 시 적재
- `insert`: 신규 행
- `update`: 행 수정
- `delete`: 행 삭제

**Control Record** — 테이블 정의/변경 이벤트:

```json
{
  "control": {
    "table-def": {
      "columns": {
        "id": { "type": "INT32", "nullable": false },
        "fname": { "type": "WSTRING", "length": 255, "nullable": true }
      },
      "primary-key": ["id"]
    }
  },
  "metadata": {
    "record-type": "control",
    "operation": "create-table",
    "schema-name": "public",
    "table-name": "some_table"
  }
}
```

Control Record의 `operation`으로는 `create-table`, `drop-table`, `add-column`, `drop-column` 등이 있다.

### MSK Serverless

Kafka 브로커 관리 없이 사용할 수 있는 서버리스 Kafka다. 스케일링이나 패치를 신경 쓸 필요가 없다는 장점이 있지만, **Schema Registry를 지원하지 않는다**. 그래서 메시지 스키마 검증이나 진화(evolution)를 Kafka 레벨에서 처리할 수 없고, consumer 코드에서 직접 대응해야 한다.

### Dagster + Spark (Micro-batch)

Dagster가 5~10분 주기로 Spark job을 트리거하고, Spark가 MSK에서 메시지를 읽어 Iceberg에 쓴다.

흐름을 정리하면:

1. MSK에서 지정된 offset 이후의 메시지를 batch로 읽는다
2. JSON 메시지를 파싱하고 `metadata.operation` 기준으로 분류한다
3. Iceberg 테이블에 MERGE 연산을 수행한다

MERGE 전략은 `metadata.operation` 값에 따라 다르게 동작한다:

- `load` / `insert` / `update`: 대상 테이블에 PK 기준으로 매칭되는 행이 있으면 update, 없으면 insert
- `delete`: 매칭되는 행을 delete

### Apache Iceberg

최종 sink 테이블이다. Iceberg를 선택한 이유는 MERGE 지원, 스키마 진화, 타임 트래블 등 CDC 파이프라인에 필요한 기능을 갖추고 있기 때문이다.

---

## 스키마 변경 처리

CDC 파이프라인에서 가장 까다로운 부분 중 하나가 소스 DB의 스키마 변경이다. 컬럼이 추가되거나 삭제되면 MSK에 들어오는 메시지 구조가 달라지는데, 이걸 다운스트림에서 어떻게 처리할 것인가가 문제다.

### Schema Registry가 없는 상황

MSK Serverless를 사용하고 있어서 Schema Registry를 쓸 수 없다. Schema Registry가 있었다면 메시지 스키마의 호환성 검증이나 진화를 Kafka 레벨에서 관리할 수 있었겠지만, 현재 구조에서는 consumer 코드에서 직접 대응해야 한다.

### 코드 레벨에서의 처리

DMS는 소스 DB에 스키마 변경이 발생하면 Control Record로 변경 사항을 알려주긴 하지만, 실제로 Control Record를 파싱해서 활용하지는 않는다. 대신 Data Record의 `data` 필드에 새로운 컬럼이 추가되거나 기존 컬럼이 사라지는 것을 직접 감지하는 방식으로 처리한다.

Spark에서 이를 처리하는 접근 방식은 다음과 같다:

1. **JSON 파싱 시 스키마를 고정하지 않는다** — `schema_of_json`이나 하드코딩된 스키마 대신, 들어오는 데이터의 스키마와 캐싱해둔 현재 Iceberg 테이블의 스키마를 비교한다.
2. **Iceberg 테이블과 메시지 스키마를 비교한다** — 메시지에 새로운 컬럼이 있으면 Iceberg 테이블에 컬럼을 추가하고, 사라진 컬럼은 null로 채운다.
3. **MERGE 시 존재하는 컬럼만 업데이트한다** — 메시지에 포함된 컬럼만 대상으로 MERGE를 수행한다.

이 방식이 깔끔하진 않지만, Schema Registry 없이 스키마 변경에 대응할 수 있는 현실적인 방법이었다.

---

## 운영 경험 & 회고

### 코드 레벨 제어의 장점

코드에서 직접 처리하면서 스키마 변경 외에도 좋아진 점이 있다. 이전에 Kafka Connect를 쓸 때는 같은 PK에 대해 반복적인 update가 들어오면 제대로 반영이 안 되거나 중복 행이 생기는 문제를 겪었다. 커넥터 내부의 처리 로직을 제어할 수 없으니 원인 파악도 어려웠다.

코드 레벨에서 MERGE를 직접 수행하면서 이런 문제가 사라졌다. 같은 PK에 대한 여러 이벤트가 하나의 micro-batch에 들어와도, `metadata.timestamp`(DMS 타임스탬프)를 기준으로 가장 최신 상태만 남기도록 처리할 수 있다. 추상화된 커넥터에 맡기는 편의성을 포기한 대신, 데이터 정합성에 대한 확신을 얻었다.

### 모니터링

Dagster의 모든 asset에 post hook으로 Sentry를 연동해두었다. asset 실행 실패 시 Sentry로 알림이 가기 때문에, 파이프라인 어느 지점에서 문제가 발생했는지 빠르게 파악할 수 있다.

### 정합성 vs 신선도

이 파이프라인은 데이터 신선도(freshness)보다 **정합성(correctness)에 초점**을 맞춰 설계했다. 5~10분 주기의 micro-batch를 선택한 것도, DMS 타임스탬프 기반으로 중복 이벤트를 정리하고 MERGE를 직접 수행하는 것도 모두 정합성을 우선한 결과다.

만약 fresh data가 최우선 요구사항이었다면 아키텍처가 달라졌을 것이다. Spark Structured Streaming으로 상시 구동되는 애플리케이션을 띄우거나, 이전처럼 Kafka Connect를 써서 near real-time으로 sink에 바로 쓰는 방식을 택했을 것이다. 다만 그 경우 앞서 언급한 중복 행이나 스키마 변경 같은 문제를 다른 방식으로 해결해야 한다.
