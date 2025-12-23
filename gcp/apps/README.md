# GCP Apps Directory

ArgoCD App-of-Apps 패턴의 Application 정의 파일들

## 파일 구조

| 파일 | 설명 |
|------|------|
| `platform-apps.yaml` | Platform 컴포넌트 (External Secrets, ArgoCD Ingress) |
| `petclinic-app.yaml` | PetClinic 애플리케이션 |

## Sync Wave 순서

```
Wave 1  → platform-infra (External Secrets)
Wave 10 → platform-ingress (ArgoCD Ingress)
Wave 15 → PetClinic Application
```

> **Note**: GKE는 ALB Controller, Karpenter, Metrics Server 기능을 기본 제공
>
> 상세 설정은 루트 [README.md](../../README.md) 참조
