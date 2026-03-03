# PROTOTYNE_manifest

## 인프라 개요

기존 EC2 기반 단일 인스턴스 구성에서 **Amazon EKS(Elastic Kubernetes Service)** 기반으로 전환한 프로젝트입니다.

### AS-IS → TO-BE

| 항목 | AS-IS (EC2) | TO-BE (EKS) |
|------|-------------|-------------|
| 배포 방식 | SSH + 수동 실행 | `kubectl apply -k` |
| 헬스체크 | 없음 | Readiness / Liveness Probe |
| 스케일 아웃 | 수동 인스턴스 추가 | `replicas` 조정 |
| HTTPS 처리 | EC2 앞단 별도 구성 | ALB + ACM 통합 |
| 환경변수 | `.env` 직접 관리 | Kubernetes Secret |
| 외부 노출 | 보안그룹 + 퍼블릭 IP | ALB Ingress (단일 진입점) |

---

## 아키텍처

```
Internet
   │
   ▼
[AWS ALB]  ← HTTPS 443 (ACM 인증서), HTTP 80
   │
   ▼  (Ingress: host=prototyne.site, pathType=Prefix /)
[Ingress Controller - AWS Load Balancer Controller]
   │
   ▼
[Service: ClusterIP, port 80 → targetPort 8080]
   │
   ▼
[Pod: live-server container, containerPort 8080]
   │
   └── envFrom: live-server-secrets (Kubernetes Secret)
```

**ClusterIP를 선택한 이유**
- 파드를 직접 퍼블릭하게 노출하지 않고, 외부 트래픽은 반드시 Ingress → Service → Pod 경로를 거치도록 강제
- `LoadBalancer` 타입은 서비스마다 ALB가 생성되어 비용이 증가하므로, Ingress 하나로 라우팅을 집중시키는 구조 선택

---

## Kubernetes 리소스 구성

```
k8s/
├── deployment.yaml     # 파드 스펙 및 컨테이너 정의
├── service.yaml        # ClusterIP 서비스 (내부 통신 전용)
├── ingress.yaml        # AWS ALB 기반 외부 노출
└── kustomization.yaml  # 리소스 묶음 관리
```

### Deployment

- `replicas: 1` — 현재 단일 파드 운영 (트래픽 증가 시 HPA 연동 예정)
- `envFrom.secretRef` — Secret 오브젝트 전체를 환경변수로 주입, manifest에 민감 정보 미포함
- 이미지 태그를 명시적으로 고정 (`1.0.12`) — `latest` 사용 지양으로 배포 재현성 확보

### Service

- `type: ClusterIP` — 클러스터 내부 전용 엔드포인트
- `port: 80 → targetPort: 8080` — 서비스 레이어와 컨테이너 포트를 분리하여 애플리케이션 포트 변경 시 서비스 스펙에 영향 없음

### Ingress

- `alb.ingress.kubernetes.io/scheme: internet-facing` — 퍼블릭 ALB 생성
- `listen-ports: HTTP 80 + HTTPS 443` — 두 포트 동시 리슨, HTTP → HTTPS 리다이렉트 정책과 병행 가능
- `certificate-arn` — ACM 인증서를 ALB에 직접 오프로드, 파드 내부에서 TLS 처리 불필요
- `ingressClassName: alb` — AWS Load Balancer Controller 명시적 지정

### Kustomize

- `kubectl apply -k` 한 번으로 세 리소스를 원자적으로 적용
- 추후 `overlays/` 구조 확장 시 환경별(dev/staging/prod) 설정 분기 가능

---

## 배포

### 사전 조건

- `kubectl` 설치 및 EKS 클러스터 kubeconfig 구성
- AWS Load Balancer Controller 클러스터에 설치 완료 (IRSA 설정 포함)
- ACM 인증서 ARN 발급

### Secret 생성

```bash
kubectl create secret generic live-server-secrets \
  --from-literal=KEY=VALUE
```

### 배포 적용

`ingress.yaml`의 `PLACEHOLDER-TO-BE-PATCHED`를 실제 ACM ARN으로 교체 후:

```bash
kubectl apply -k k8s/
```

### 상태 확인

```bash
# 파드 상태
kubectl get pods -l app=live-server

# ALB 주소 확인 (ADDRESS 컬럼)
kubectl get ingress live-server-ingress

# 파드 로그
kubectl logs -l app=live-server --tail=100
```

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| Container | Docker |
| Orchestration | Amazon EKS (Kubernetes) |
| Ingress | AWS Load Balancer Controller (ALB) |
| TLS | AWS Certificate Manager (ACM) |
| 설정 관리 | Kustomize |
| 민감정보 | Kubernetes Secret |

---

## 개선 예정

- [ ] Readiness / Liveness Probe 추가로 비정상 파드 자동 교체
- [ ] HPA(Horizontal Pod Autoscaler) 연동으로 트래픽 기반 자동 스케일 아웃
- [ ] Kustomize `overlays/` 구조 도입으로 환경별 설정 분리
- [ ] PodDisruptionBudget 설정으로 롤링 업데이트 중 가용성 보장

