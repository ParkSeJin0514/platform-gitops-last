# 🚀 Platform GitOps (Multi-Cloud: AWS / GCP)

ArgoCD App-of-Apps 패턴을 사용한 멀티 클라우드 GitOps 매니페스트

## 🏛️ 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ArgoCD App-of-Apps Pattern                        │
├─────────────────────────────────┬───────────────────────────────────┤
│         AWS (Primary)           │          GCP (DR/Secondary)       │
├─────────────────────────────────┼───────────────────────────────────┤
│  aws/apps/                      │  gcp/apps/                        │
│  ├── platform-apps.yaml         │  ├── platform-apps.yaml           │
│  └── petclinic-app.yaml         │  └── petclinic-app.yaml           │
├─────────────────────────────────┼───────────────────────────────────┤
│  aws/platform/                  │  gcp/platform/                    │
│  ├── alb-controller/            │  ├── external-secrets/            │
│  ├── efs-csi-driver/            │  └── argocd-ingress/              │
│  ├── external-secrets/          │                                   │
│  ├── karpenter/                 │                                   │
│  ├── karpenter-config/          │                                   │
│  └── argocd-ingress/            │                                   │
└─────────────────────────────────┴───────────────────────────────────┘
```

## 📁 디렉토리 구조

```
platform-gitops-last/
├── aws/                          # AWS Platform Components
│   ├── apps/
│   │   ├── platform-apps.yaml   # Platform ApplicationSet
│   │   └── petclinic-app.yaml   # PetClinic Application
│   └── platform/
│       ├── alb-controller/      # AWS ALB Controller
│       ├── efs-csi-driver/      # AWS EFS CSI Driver
│       ├── external-secrets/    # AWS Secrets Manager
│       ├── karpenter/           # Karpenter Controller
│       ├── karpenter-config/    # NodePool, EC2NodeClass
│       └── argocd-ingress/      # ALB Ingress
│
├── gcp/                          # GCP Platform Components
│   ├── apps/
│   │   ├── platform-apps.yaml   # Platform Applications
│   │   └── petclinic-app.yaml   # PetClinic Application
│   └── platform/
│       ├── external-secrets/    # GCP Secret Manager
│       └── argocd-ingress/      # GKE Ingress
│
└── applications/                 # 공통 애플리케이션 설정
    └── petclinic/
        └── external-secret.yaml # DB Secret 참조
```

## ☁️ AWS vs GCP 컴포넌트 비교

| 컴포넌트 | AWS | GCP | 비고 |
|---------|-----|-----|------|
| **Load Balancer** | ALB Controller | GKE Ingress (GCE) | GKE 기본 제공 |
| **Auto Scaling** | Karpenter | - | GKE Autopilot 내장 |
| **Storage** | EFS CSI Driver | - | 필요시 Filestore |
| **Secrets** | External Secrets (AWS SM) | External Secrets (GCP SM) | Workload Identity |
| **Ingress** | ALB Ingress | GKE Ingress | 각 클라우드 네이티브 |
| **Monitoring** | kube-prometheus-stack | kube-prometheus-stack | petclinic-gitops에서 관리 |

## 🎯 Karpenter 설정

### NodePool 구성

```yaml
requirements:
  - key: kubernetes.io/arch
    values: [amd64]
  - key: karpenter.sh/capacity-type
    values: [spot, on-demand]       # Spot 우선, On-Demand fallback
  - key: karpenter.k8s.aws/instance-category
    values: [t]                      # t 시리즈 (비용 효율)
  - key: karpenter.k8s.aws/instance-size
    values: [medium, large, xlarge, 2xlarge]
  - key: topology.kubernetes.io/zone
    values: [ap-northeast-2a, ap-northeast-2b]  # 멀티 AZ 분산
```

### EC2NodeClass 구성

| 설정 | 값 | 설명 |
|------|-----|------|
| AMI | AL2@latest | Amazon Linux 2 EKS 최적화 (AL2023은 nodeadm 호환성 이슈) |
| EBS | gp3, 30GB | 암호화 활성화 |
| IMDS | IMDSv2 필수 | 보안 강화 |

### 서브넷 IP 관리

> **주의**: /24 서브넷에서는 Secondary IP 모드 사용 권장

| 모드 | 서브넷 크기 | 노드당 최대 Pod |
|------|-------------|-----------------|
| Secondary IP | /24 이상 | 인스턴스별 상이 (t3.medium: 17) |
| Prefix Delegation | /20 이상 | 110 |

## 📊 Sync Wave 순서

### AWS
```
Wave 1  → ALB Controller, EFS CSI Driver, External Secrets
Wave 5  → Karpenter Controller
Wave 6  → Karpenter Config (NodePool, EC2NodeClass)
Wave 10 → ArgoCD Ingress
Wave 15 → PetClinic Application
```

### GCP
```
Wave 1  → External Secrets
Wave 10 → ArgoCD Ingress
Wave 15 → PetClinic Application
```

> **Note**: kube-prometheus-stack은 petclinic-gitops에서 관리 (AWS와 동일한 구조)

## 🔐 External Secrets 설정

### AWS (IRSA)
```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME"
```

### GCP (Workload Identity)
```yaml
serviceAccount:
  annotations:
    iam.gke.io/gcp-service-account: "SA_NAME@PROJECT_ID.iam.gserviceaccount.com"
```

## ⚙️ ArgoCD 설정

ArgoCD Bootstrap 시 클라우드에 따라 다른 gitops_path 사용:

| Cloud | GitOps Path | 설명 |
|-------|-------------|------|
| AWS | `aws/apps` | AWS 플랫폼 컴포넌트 + PetClinic |
| GCP | `gcp/apps` | GCP 플랫폼 컴포넌트 + PetClinic |

## 🚀 사용 방법

### 1. ArgoCD 접속
```bash
# AWS
kubectl get ingress -n argocd argocd-server

# GCP
kubectl get ingress -n argocd argocd-server
```

### 2. 초기 비밀번호 확인
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3. Application 동기화

ArgoCD는 자동으로 `platform-apps.yaml`과 `petclinic-app.yaml`을 동기화합니다.

## 🔄 DR 시나리오

### AWS → GCP Failover

1. GCP Terraform Apply (foundation → compute → bootstrap)
2. ArgoCD가 자동으로 `gcp/apps/*` 동기화
3. DNS 또는 Global Load Balancer를 GCP로 전환
4. PetClinic이 GCP Cloud SQL에 접근 (각 클라우드 별도 DB)

### GCP → AWS Failback

1. AWS 복구 확인
2. DNS를 AWS로 전환
3. GCP 리소스 정리 (선택사항)

## 🐳 컨테이너 레지스트리

| 클라우드 | 레지스트리 | 리전 |
|---------|-----------|------|
| **AWS** | ECR | ap-northeast-2 |
| **GCP** | Artifact Registry | asia-northeast3 |

```bash
# AWS ECR
946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic-msa/petclinic-*

# GCP Artifact Registry
asia-northeast3-docker.pkg.dev/kdt2-final-project-t1/petclinic-msa/petclinic-*
```

## 🔗 관련 저장소

| 저장소 | 설명 |
|--------|------|
| **platform-dev-last** | Terraform/Terragrunt IaC 코드 (EKS, GKE, VPC) |
| **petclinic-gitops** | PetClinic 애플리케이션 Kubernetes 매니페스트 |
| **petclinic-dev** | PetClinic 소스 코드 + Multi-Cloud CI/CD |
