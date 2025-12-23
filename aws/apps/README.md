# AWS Apps Directory

ArgoCD App-of-Apps 패턴의 Application 정의 파일들

## 파일 구조

| 파일 | 설명 |
|------|------|
| `platform-apps.yaml` | Platform 컴포넌트 (ALB Controller, Karpenter 등) |
| `petclinic-app.yaml` | PetClinic 애플리케이션 |

## Sync Wave 순서

```
Wave 1  → platform-infra (ALB Controller, EFS CSI Driver, External Secrets, Metrics Server)
Wave 5  → Karpenter Controller
Wave 6  → Karpenter Config (NodePool, EC2NodeClass)
Wave 10 → platform-ingress (ArgoCD Ingress)
Wave 15 → PetClinic Application
```

> 상세 설정은 루트 [README.md](../../README.md) 참조
