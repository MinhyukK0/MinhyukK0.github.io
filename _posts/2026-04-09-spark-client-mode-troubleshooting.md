---
title: "Spark Client Mode on K8s - 드라이버 리소스 트러블슈팅"
date: 2026-04-09 14:00:00 +0900
categories: [Troubleshooting]
tags: [spark, kubernetes, dagster, iceberg, client-mode]
---

## 배경

Dagster pod이 Spark driver(client mode)로 동작하며, K8s에 executor pod을 동적으로 생성하는 구조다. Iceberg 테이블에 대해 메인터넌스(expire_snapshots, remove_orphan_files, rewrite_data_files 등)를 수행하는 Job을 대용량 테이블(1TB+) 기준으로 테스트하던 중 에러가 발생했다.

## 에러 현상

두 번째 asset(`iceberg_remove_orphan_files`) 실행 시 SparkSession 생성 단계에서 실패했다.

```
py4j.protocol.Py4JError: An error occurred while calling None.org.apache.spark.sql.classic.SparkSession
```

첫 번째 asset(`iceberg_expire_snapshots`)은 정상 완료되었고, 두 번째 asset에서 SparkSession을 새로 생성하려다 실패하는 패턴이었다.

## 원인 분석

### 1. Multiprocess Executor 구조

Dagster의 multiprocess executor는 각 asset을 **별도 프로세스**에서 실행한다. 4개 asset이 순차적 deps 체인으로 연결되어 있지만, 매번 새 프로세스에서 새 SparkSession(JVM)을 생성한다.

```
iceberg_expire_snapshots → iceberg_remove_orphan_files → iceberg_rewrite_data_files → iceberg_rewrite_position_delete_files
```

각 프로세스가 JVM을 개별로 띄우므로 driver pod의 메모리 사용량이 누적될 수 있다.

### 2. Pod 리소스 설정 불일치

Run launcher pod의 리소스 설정을 확인한 결과:

| 항목 | 값 |
|---|---|
| memory request | 4Gi |
| memory limit | 8Gi |

**request와 limit의 차이**: Karpenter(노드 오토스케일러)는 request 기준으로 노드를 할당한다. request(4Gi)와 limit(8Gi)이 다르면 노드가 overcommit 상태가 되고, 리소스 압박 시 eviction이 발생할 수 있다.

### 3. 실제 K8s 이벤트 확인

```bash
kubectl get events -n dagster --sort-by='.lastTimestamp' | grep -i -E "oom|kill|evict|fail"
```

```
FailedCreatePodSandBox  — aws-cni plugin failed (네트워크 설정 실패)
Evicted                 — Underutilized (노드 eviction)
```

OOMKilled은 아니었지만, 노드 스케줄링 과정에서 CNI 네트워크 설정 실패와 eviction이 발생했다. request/limit 불일치로 Karpenter가 부적절한 노드에 배치한 것이 원인으로 추정된다.

## 해결

Run launcher pod의 memory request와 limit을 모두 8Gi로 통일하여, Karpenter가 처음부터 충분한 노드에 배치하도록 변경했다.

```hcl
resources = {
    requests = {
        cpu    = "500m"
        memory = "8Gi"  # 4Gi → 8Gi
    }
    limits = {
        memory = "8Gi"
    }
}
```

## 교훈

### Client Mode에서 드라이버 리소스 관리

Spark client mode에서는 제출한 pod 자체가 driver가 된다. Cluster mode와 달리 driver 리소스를 별도로 할당할 수 없고, **pod의 리소스 설정이 곧 driver의 리소스 한계**다.

### K8s request/limit 설정 원칙

| 상황 | 권장 |
|---|---|
| 일반적인 웹 서버 | request < limit (버스트 허용) |
| Spark driver처럼 JVM 고정 할당 | **request = limit** (예측 가능한 스케줄링) |

JVM은 힙 메모리를 미리 할당하므로 실제 사용량이 거의 고정이다. request와 limit을 다르게 설정하면 Karpenter가 실제보다 적은 리소스로 노드를 배치하고, 런타임에 문제가 발생한다.

### Multiprocess Executor의 메모리 영향

Dagster multiprocess executor는 각 step을 별도 프로세스에서 실행한다. Spark 기반 asset이 순차 실행이더라도 이전 프로세스의 JVM이 완전히 정리되기 전에 다음 JVM이 시작될 수 있다. 메모리 여유를 충분히 확보하거나, in_process executor를 사용해 SparkSession을 재사용하는 것이 대안이다.
