---
title: "Kubernetes HA — Tier 기반 환경별 차등 전략"
description: "로컬(Kind)과 프로덕션(AWS) 환경에서 서비스 Tier에 따라 replica, anti-affinity, HPA, PDB를 차등 적용한 Kustomize overlay 설계 기록"
author: laze
date: 2026-03-12 13:00:00 +0900
categories: [Dev, Kubernetes]
tags: [Kubernetes, Kustomize, HA]
---

## 문제

마이크로서비스 17개를 Kubernetes에 배포해야 합니다. 모든 서비스에 동일한 HA 설정을 적용하면 리소스가 낭비되고, 환경(로컬 vs 프로덕션)에 따라 요구사항도 다릅니다. 로컬 Kind 클러스터에서 api-gateway를 3개 replica로 띄울 이유는 없지만, 프로덕션에서는 1개면 SPOF(Single Point of Failure)입니다.

## 설계: Tier 기반 차등 정책

장애 시 영향 범위와 트래픽 패턴을 기준으로 서비스를 4개 Tier로 분류했습니다. 전체 서비스가 의존하는 게이트웨이·인증은 Critical, 매출과 직결되는 커머스·메인 UI는 High, 나머지는 독립적으로 장애를 격리할 수 있으므로 Standard 또는 Frontend로 구분합니다.

| Tier | 서비스 | 기준 |
|------|--------|------|
| **Critical** | api-gateway, auth-service | 장애 시 전체 서비스 불가 |
| **High** | shopping-service, shopping-seller-service, portal-shell | 핵심 비즈니스 기능 |
| **Standard** | blog, notification, drive, prism, chatbot, settlement | 장애 시 일부 기능 불가 |
| **Frontend** | 6개 frontend 앱 | 정적 자산 서빙 |

### Tier별 환경 설정

| 설정 | Critical (Kind/AWS) | High (Kind/AWS) | Standard | Frontend (Kind/AWS) |
|------|:---:|:---:|:---:|:---:|
| **Replicas** | 1 / 3 | 1 / 2 | 1 / 1 | 1 / 2 |
| **Anti-Affinity** | preferred / required | preferred / preferred | 없음 | 없음 / preferred |
| **HPA** | O / O | O / O | X | X |
| **PDB** | O / O | O / O | X | X / O |

Standard Tier는 트래픽 급변이 없거나(notification, settlement은 배치성), 정적 자산만 서빙하므로(frontend) HPA 대신 고정 replica를 사용합니다.

## Kustomize Overlay 구조

```
k8s/
├── base/                    # 공통 설정 (namespace, secrets)
├── services/                # 서비스별 Deployment, Service
├── infrastructure/          # DB, Kafka, Redis, Monitoring
└── overlays/
    ├── kind/                # 로컬 개발 환경
    │   ├── kustomization.yaml
    │   ├── patches/         # replicas, affinity, resources
    │   ├── hpa/             # HPA 6개
    │   ├── pdb/             # PDB 4개
    │   └── metrics-server.yaml
    └── aws/                 # 프로덕션 환경
        ├── kustomization.yaml
        ├── patches/         # replicas, affinity, resources
        ├── hpa/             # HPA 5개
        ├── pdb/             # PDB 4개
        ├── configmap.yaml   # AWS managed service 엔드포인트
        └── ingress-alb.yaml # ALB Ingress
```

### Replica Patch

```yaml
# overlays/kind/patches/replicas.yaml — 모든 서비스 1 replica
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 1

# overlays/aws/patches/replicas.yaml — Tier별 차등
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3    # Critical
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopping-service
spec:
  replicas: 2    # High
```

### Anti-Affinity

같은 서비스의 Pod가 동일 노드에 몰리는 것을 방지합니다. Tier에 따라 강도를 다르게 설정합니다.

```yaml
# Kind (Critical Tier): preferred — 가능하면 분산하되, 노드가 부족하면 같은 노드 허용
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: api-gateway
                topologyKey: kubernetes.io/hostname

# AWS (Critical Tier): required — 반드시 다른 노드에 배치
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: api-gateway
              topologyKey: kubernetes.io/hostname
```

Kind에서 `required`를 쓰면 노드가 1~3개인 로컬 환경에서 Pod가 Pending 상태에 빠질 수 있으므로, `preferred`로 완화합니다.

### HPA (Horizontal Pod Autoscaler)

Critical/High Tier에 적용합니다. CPU threshold 70%는 스케일 업 시 약 30%의 헤드룸을 남겨 새 Pod가 Ready 상태가 되기 전까지의 트래픽 버퍼를 확보하기 위한 값이며, Kubernetes 공식 문서에서도 권장하는 일반적인 기준입니다. Memory 80%는 JVM 힙 특성상 GC 후 사용률이 크게 변동하므로 CPU보다 여유를 둡니다.

```yaml
# Kind — api-gateway HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 1
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30     # 30초 안정화 후 스케일 업
      policies:
        - type: Pods
          value: 2                        # 60초마다 최대 2개 추가
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300    # 5분 안정화 후 스케일 다운
      policies:
        - type: Pods
          value: 1                        # 120초마다 최대 1개 제거
          periodSeconds: 120
```

스케일 업은 빠르게(30초 안정화, 2개씩), 스케일 다운은 보수적으로(5분 안정화, 1개씩) 설정했습니다. 트래픽이 잠시 줄었다가 다시 올라오는 상황에서 불필요하게 Pod를 제거하는 것을 방지합니다.

AWS에서는 Critical Tier의 min/max를 더 높게 설정합니다 (3/10).

### PDB (Pod Disruption Budget)

```yaml
# Kind — minAvailable: 1 (최소 1개 유지)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: api-gateway

# AWS — minAvailable: 2 (3개 중 최소 2개 유지)
spec:
  minAvailable: 2
```

### Kind vs AWS 인프라 차이

| 인프라 | Kind | AWS |
|--------|------|-----|
| PostgreSQL | 자체 Pod | Amazon RDS |
| Kafka | 자체 Pod | Amazon MSK |
| Redis | 자체 Pod | ElastiCache |
| MongoDB | 자체 Pod | DocumentDB |
| Metrics | Metrics Server Pod | CloudWatch |
| Ingress | NodePort | ALB Ingress |

Kind overlay에는 인프라 Pod와 Metrics Server가 포함되고, AWS overlay에는 관리형 서비스 엔드포인트가 ConfigMap으로 제공됩니다.

```yaml
# AWS ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-config
data:
  POSTGRES_HOST: portal-universe-rds.xxxx.ap-northeast-2.rds.amazonaws.com
  KAFKA_BOOTSTRAP_SERVERS: b-1.portal-msk.xxxx.kafka.ap-northeast-2.amazonaws.com:9092,...
  REDIS_HOST: portal-redis.xxxx.cache.amazonaws.com
```

## 교훈

**1. 모든 서비스에 동일 HA를 적용하는 것은 낭비다**

Standard Tier 서비스에 3 replica + PDB + HPA를 설정하면 리소스만 소모됩니다. 서비스 중요도를 분류하고, Tier별로 차등 정책을 적용하는 것이 효율적입니다.

**2. 로컬 환경은 프로덕션의 축소판이 아니라 별도 설계가 필요하다**

Kind에서 `required` anti-affinity를 쓰면 Pod가 Pending에 빠지고, 프로덕션용 리소스 limit을 그대로 쓰면 OOM이 발생합니다. 환경별로 overlay를 분리하는 것이 Kustomize의 본래 용도입니다.

**3. PDB는 replica 수와 함께 설계해야 한다**

`minAvailable: 2`인데 replica가 2개이면 drain이 영원히 완료되지 않습니다. PDB 값은 항상 replica 수보다 최소 1 작게 설정해야 롤링 업데이트와 노드 유지보수가 정상 동작합니다.
