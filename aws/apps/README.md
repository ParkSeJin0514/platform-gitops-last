# AWS Apps Directory

ArgoCD App-of-Apps 패턴의 Application 정의 파일들

## 파일 구조

| 파일 | 설명 |
|------|------|
| `platform-apps.yaml` | Platform 컴포넌트 (ALB Controller, Karpenter 등) |
| `petclinic-app.yaml` | PetClinic 애플리케이션 |

## 동작 방식

1. Terraform Bootstrap에서 `root-app` 생성
2. `root-app`이 이 디렉토리의 Application 파일들을 감지
3. ArgoCD가 각 Application을 자동 배포

## Sync Wave 순서

```
Wave 1  → ALB Controller, EFS CSI Driver, External Secrets
Wave 5  → Karpenter Controller
Wave 6  → Karpenter Config (NodePool, EC2NodeClass)
Wave 10 → ArgoCD Ingress
Wave 15 → PetClinic Application
```