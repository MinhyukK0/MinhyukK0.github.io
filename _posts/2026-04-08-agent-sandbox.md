---
title: "Agent Sandbox - AI 에이전트를 위한 격리된 코드 실행 환경"
date: 2026-04-08 14:00:00 +0900
categories: [Tech Exploration]
tags: [agent-sandbox, kubernetes, gvisor, ai-agent, eks]
---

## Agent Sandbox란?

AI 에이전트가 코드를 실행할 때, 호스트 시스템에 직접 접근하면 보안 위험이 크다. Agent Sandbox는 Kubernetes 위에서 **격리된 코드 실행 환경(Sandbox)**을 제공하는 프로젝트다. [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox)로 공개되어 있다.

핵심 아이디어:
- AI 에이전트가 실행하는 코드를 **gVisor 런타임으로 격리**하여 호스트 커널을 보호한다
- Sandbox를 **Kubernetes CRD로 선언적으로 관리**한다
- 미리 워밍된 Sandbox 풀(WarmPool)로 **빠른 시작 시간**을 보장한다

## 아키텍처

```
Client (AI Agent)
    │
    ▼
Router (Deployment, 2 replicas)
    │ ← 요청을 적절한 Sandbox로 라우팅
    ▼
Controller (StatefulSet, 1 replica)
    │ ← Sandbox 리소스 생애주기 관리
    ├── Sandbox Pod (gVisor)  ← 격리된 코드 실행
    ├── Sandbox Pod (gVisor)
    └── Sandbox Pod (gVisor)
```

### 구성 요소

| 컴포넌트 | 타입 | 역할 |
|---|---|---|
| **Controller** | StatefulSet | Sandbox CRD를 감시하고 Pod 생성/삭제를 관리 |
| **Router** | Deployment | 클라이언트 요청을 해당 Sandbox Pod로 라우팅 |
| **Sandbox Pod** | Pod (gVisor) | 실제 코드가 실행되는 격리 환경 |

### Namespace 분리

- `agent-sandbox-system`: Controller, Router 등 관리 컴포넌트
- `agent-sandbox`: 실제 Sandbox Pod 인스턴스

관리 영역과 실행 영역을 분리하여 RBAC 범위를 최소화한다.

## CRD (Custom Resource Definitions)

Agent Sandbox는 4개의 CRD를 정의한다.

### Sandbox

개별 Sandbox 인스턴스다. Controller가 이 리소스를 감시하고 해당하는 Pod를 생성한다.

```yaml
apiVersion: agents.x-k8s.io/v1alpha1
kind: Sandbox
metadata:
  name: my-sandbox
  namespace: agent-sandbox
spec:
  # Sandbox 스펙 정의
```

### SandboxTemplate

Sandbox의 템플릿이다. 재사용 가능한 Sandbox 스펙을 정의해두면 SandboxClaim에서 참조한다.

### SandboxClaim

SandboxTemplate을 참조하여 Sandbox를 요청한다. PVC(PersistentVolumeClaim)가 PV를 요청하는 패턴과 유사하다.

```
SandboxTemplate (스펙 정의) ← SandboxClaim (요청) → Sandbox (인스턴스 생성)
```

### SandboxWarmPool

SandboxTemplate 기반으로 **미리 Sandbox를 생성해두는 풀**이다. AI 에이전트가 Sandbox를 요청하면 이미 준비된 인스턴스를 즉시 할당하여 콜드 스타트를 줄인다.

## gVisor 런타임

Agent Sandbox의 격리는 **gVisor(runsc)**로 구현된다. gVisor는 Google이 개발한 컨테이너 런타임으로, 애플리케이션의 시스템 콜을 사용자 공간에서 인터셉트한다.

```
일반 컨테이너:     App → 시스템 콜 → 호스트 커널
gVisor 컨테이너:   App → 시스템 콜 → gVisor(Sentry) → 제한된 호스트 커널 접근
```

- 호스트 커널에 직접 시스템 콜을 보내지 않으므로 **커널 취약점으로부터 보호**
- VM보다 가볍고, 일반 컨테이너보다 강한 격리
- Kubernetes의 RuntimeClass로 gVisor를 지정하여 특정 Pod만 gVisor로 실행 가능

EKS에서는 gVisor가 설치된 Custom AMI를 별도 노드 그룹으로 구성하여 Sandbox Pod를 해당 노드에 스케줄링한다.

## 실제 적용

### EKS 배포 구성

Terraform으로 EKS 클러스터에 배포했다. Controller와 Router는 ops 노드에, Sandbox Pod는 gVisor 노드에 스케줄링된다.

```
EKS Cluster
├── ops 노드 그룹
│   ├── Controller (StatefulSet)
│   └── Router (Deployment x 2)
│
└── gVisor 노드 그룹 (Custom AMI)
    ├── Sandbox Pod
    ├── Sandbox Pod
    └── ...
```

### Controller 배포

공식 릴리스 매니페스트를 vendoring하여 사용한다. `--extensions` 플래그로 SandboxClaim, SandboxTemplate, SandboxWarmPool을 활성화한다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: agent-sandbox-controller
  namespace: agent-sandbox-system
spec:
  replicas: 1
  template:
    spec:
      containers:
      - args:
        - --extensions
        image: registry.k8s.io/agent-sandbox/agent-sandbox-controller:v0.1.0
      nodeSelector:
        node-group: ops
```

### Router 배포

Router는 공식 매니페스트에 포함되어 있지 않아 별도로 구성했다. 클라이언트 요청을 Sandbox Pod로 라우팅하는 역할을 한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sandbox-router
  namespace: agent-sandbox-system
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: router
          image: <ECR>/agent-sandbox:router-0.1.10
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
```

### 접근 경로

클러스터 내부에서는 Service로 직접 접근하고, 외부에서는 Istio Internal Gateway를 통해 VPN 접속 후 접근한다.

```
# 클러스터 내부 (권장)
Pod → sandbox-router-svc.agent-sandbox-system:8080

# 외부 (VPN 필요)
VPN → Internal NLB → Istio Internal Gateway → VirtualService → Router
```

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: agent-sandbox-internal
spec:
  hosts:
    - "agent-sandbox.quantit.ai"
  gateways:
    - istio-ingress/internal-gateway
  http:
    - route:
        - destination:
            host: sandbox-router-svc.agent-sandbox-system.svc.cluster.local
            port:
              number: 8080
```

### IRSA 연동 (Athena/S3)

Sandbox Pod에서 AWS 리소스에 접근해야 하는 경우, IRSA로 권한을 부여한다. `sandbox-runner` ServiceAccount를 통해 Athena 쿼리 실행과 S3 접근이 가능하다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sandbox-runner
  namespace: agent-sandbox
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/agent-sandbox-pod
```

부여된 권한:
- **Athena**: 쿼리 실행, 결과 조회, Workgroup/Catalog 접근
- **Glue**: 데이터베이스/테이블 메타데이터 조회
- **S3**: 쿼리 결과 버킷 읽기/쓰기, 데이터 소스 버킷 읽기

## 일반 컨테이너 vs Agent Sandbox

| 항목 | 일반 컨테이너 | Agent Sandbox (gVisor) |
|---|---|---|
| 격리 수준 | namespace/cgroup (커널 공유) | gVisor가 시스템 콜 인터셉트 |
| 커널 취약점 노출 | 있음 | 최소화 |
| 시작 시간 | 빠름 | WarmPool로 보완 |
| 리소스 오버헤드 | 낮음 | gVisor 런타임 오버헤드 존재 |
| 관리 방식 | Pod 직접 관리 | CRD로 선언적 관리 |
| 적합한 용도 | 신뢰할 수 있는 워크로드 | AI 에이전트의 비신뢰 코드 실행 |

## 참고

- [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox)
- [gVisor Documentation](https://gvisor.dev/docs/)
- [Kubernetes RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
