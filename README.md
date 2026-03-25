# demo-eks-gitops

ArgoCD Day 2 GitOps 레포. Terraform이 생성한 AWS 리소스 위에서 K8s 컴포넌트를 관리.

## 구조

```
demo-eks-gitops/
├── platform/                    # ArgoCD Application 정의 (App of Apps 진입점)
│   ├── karpenter.yaml           #   → charts/karpenter/ 참조
│   ├── prometheus.yaml          #   → Helm repo 직접 참조
│   ├── grafana.yaml             #   → Helm repo 직접 참조
│   ├── loki.yaml                #   → Helm repo 직접 참조 (★ IRSA ARN 기입 필요)
│   ├── opencost.yaml            #   → Helm repo 직접 참조 (★ IRSA ARN 기입 필요)
│   ├── kafka.yaml               #   → Strimzi Helm + charts/kafka/ CR
│   └── redis.yaml               #   → Helm repo 직접 참조
│
├── charts/                      # 커스텀 Chart (Helm repo에 없는 것)
│   ├── karpenter/               #   EC2NodeClass + NodePool
│   │   ├── Chart.yaml
│   │   ├── values.yaml          #   ★ clusterName, karpenterNodeRoleName 기입
│   │   └── templates/
│   │       ├── ec2nodeclass-default.yaml
│   │       └── nodepool-default.yaml
│   └── kafka/                   #   Strimzi Kafka CR
│       ├── Chart.yaml
│       └── templates/
│           └── kafka-cluster.yaml
│
└── README.md
```

## 사전 조건

Terraform 레이어가 모두 apply 완료되어야 함:

```
1.infra       ✓ VPC, S3, ECR
2.eks-core    ✓ EKS Cluster
3.eks-node    ✓ Infra Node Group
4.eks-addon   ✓ Add-on + Karpenter + ALB Controller + ArgoCD
5.eks-bootstrap ✓ ArgoCD Projects + Root Application
```

## 정적값 기입 (1회)

terraform output으로 확인한 값을 해당 파일에 기입:

```bash
cd 2.env/1.dev/4.eks-addon
terraform output
```

| 값 | 기입 위치 |
|----|----------|
| loki_irsa_arn | platform/loki.yaml → `eks.amazonaws.com/role-arn` |
| opencost_irsa_arn | platform/opencost.yaml → `eks.amazonaws.com/role-arn` |
| loki_chunks_bucket | platform/loki.yaml → `bucketnames`, `chunks` |
| loki_ruler_bucket | platform/loki.yaml → `ruler` |
| karpenter_node_iam_role_name | charts/karpenter/values.yaml → `karpenterNodeRoleName` |

## 변경 방법

### NodePool 인스턴스 타입 변경

```yaml
# charts/karpenter/values.yaml
defaultNodePool:
  instanceTypes:
    - t3.medium
    - t3.large
    - m5.large     # ← 추가
```

```bash
git add . && git commit -m "Add m5.large to default nodepool" && git push
```

ArgoCD가 자동 Sync → 변경 반영. terraform apply 불필요.

### Prometheus retention 변경

```yaml
# platform/prometheus.yaml → helm.values
prometheus:
  prometheusSpec:
    retention: 14d     # ← 7d → 14d
```

Git push → ArgoCD 자동 반영.

### Kafka 브로커 수 변경

```yaml
# charts/kafka/templates/kafka-cluster.yaml
spec:
  replicas: 5          # ← 3 → 5
```

Git push → ArgoCD 자동 반영.
