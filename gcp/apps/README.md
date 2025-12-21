# GCP Apps Directory

ArgoCD App-of-Apps 패턴의 Application 정의 파일들

## 파일 구조

| 파일 | 설명 |
|------|------|
| `platform-apps.yaml` | Platform 컴포넌트 (External Secrets, ArgoCD Ingress) |
| `petclinic-app.yaml` | PetClinic 애플리케이션 |

## 동작 방식

1. Terraform Bootstrap에서 `root-app` 생성
2. `root-app`이 이 디렉토리의 Application 파일들을 감지
3. ArgoCD가 각 Application을 자동 배포

## Sync Wave 순서

```
Wave 1  → platform-infra (External Secrets)
Wave 10 → platform-ingress (ArgoCD Ingress)
Wave 15 → PetClinic Application
```

## ApplicationSet 구조

| ApplicationSet | 포함 컴포넌트 |
|----------------|---------------|
| `platform-infra` | External Secrets |
| `platform-ingress` | ArgoCD Ingress |

## AWS vs GCP 차이점

| 컴포넌트 | AWS | GCP | 비고 |
|---------|-----|-----|------|
| Load Balancer | ALB Controller | GKE Ingress (GCE) | GKE 기본 제공 |
| Auto Scaling | Karpenter | - | GKE Autopilot 내장 |
| Storage | EFS CSI Driver | - | 필요시 Filestore |
| Metrics Server | 별도 설치 | - | GKE Autopilot 내장 |

> **Note**: GKE Autopilot은 ALB Controller, Karpenter, Metrics Server 기능을 기본 제공하므로 별도 설치 불필요
