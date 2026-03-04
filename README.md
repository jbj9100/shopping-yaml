# Shopping Kubernetes Manifests (shopping-yaml)

Kubernetes 배포를 위한 매니페스트 저장소입니다.  
각 마이크로서비스는 **Kustomize** 기반의 `base / overlays` 구조로 관리됩니다.

---

## 📁 프로젝트 구조

```
shopping-yaml/
├── .gitlab-ci.yml                     # GitLab CI: image-digest 변경 정책 검사 (digest_guard)
├── corp-root-ca.crt                   # 사내 Root CA 인증서
│
├── shopping-backend/                  # REST API 백엔드 서비스
│   ├── CODEOWNERS
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       │       └── image-digest.yaml
│       └── prod/
│           ├── kustomization.yaml
│           └── patches/
│               └── image-digest.yaml
│
├── shopping-frontend/                 # Nginx 기반 프론트엔드 서비스
│   ├── CODEOWNERS
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       │       └── image-digest.yaml
│       └── prod/
│           ├── kustomization.yaml
│           └── patches/
│               └── image-digest.yaml
│
├── shopping-publisher/                # Kafka 이벤트 발행 서비스
│   ├── CODEOWNERS
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       │       └── image-digest.yaml
│       └── prod/
│           ├── kustomization.yaml
│           └── patches/
│               └── image-digest.yaml
│
├── shopping-websocket/                # WebSocket 실시간 통신 서비스
│   ├── CODEOWNERS
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       │       └── image-digest.yaml
│       └── prod/
│           ├── kustomization.yaml
│           └── patches/
│               └── image-digest.yaml
│
├── shopping-consumer-analytics/       # Kafka 분석 이벤트 소비 서비스
│   ├── CODEOWNERS
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       │       └── image-digest.yaml
│       └── prod/
│           ├── kustomization.yaml
│           └── patches/
│               └── image-digest.yaml
│
└── shopping-consumer-stock/           # Kafka 재고 이벤트 소비 서비스
    ├── CODEOWNERS
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── dev/
        │   ├── kustomization.yaml
        │   └── patches/
        │       └── image-digest.yaml
        └── prod/
            ├── kustomization.yaml
            └── patches/
                └── image-digest.yaml
```

---

## 🧩 서비스 개요

| 서비스 | 포트 | 역할 |
|---|---|---|
| `shopping-backend` | 8000 | REST API, DB/Redis/MinIO 연동 |
| `shopping-frontend` | 8080 | Nginx 정적 파일 서빙, Ingress 진입점 |
| `shopping-publisher` | - | DB Polling → Kafka 이벤트 발행 |
| `shopping-websocket` | 8001 | WebSocket 실시간 통신, Kafka 구독 |
| `shopping-consumer-analytics` | - | Kafka `order-events` 소비 → 분석 처리 |
| `shopping-consumer-stock` | - | Kafka `order-events` 소비 → 재고 처리 |

---

## 🏗️ Kustomize 구조

각 서비스는 동일한 패턴을 따릅니다.

```
base/                        # 공통 리소스 정의 (Deployment, Service, ConfigMap)
overlays/
  dev/                       # 개발 환경
  │   ├── kustomization.yaml
  │   └── patches/
  │       └── image-digest.yaml   # 이미지 다이제스트 고정 (Jenkins 봇 자동 갱신)
  prod/                      # 운영 환경
      ├── kustomization.yaml
      └── patches/
          └── image-digest.yaml   # 이미지 다이제스트 고정 (Jenkins 봇 자동 갱신)
```

### Kustomize 적용 명령어

```bash
# 개발 환경 배포
kubectl apply -k shopping-backend/overlays/dev

# 운영 환경 배포
kubectl apply -k shopping-backend/overlays/prod
```

---

## 🔒 CI/CD 정책 (`.gitlab-ci.yml`)

GitLab MR 파이프라인에서 **`digest_guard`** 잡이 실행됩니다.

- `overlays/(dev|prod)/patches/image-digest.yaml` 파일 변경 시 자동 검사
- **`jenkins-bot`** 계정만 해당 파일을 변경할 수 있으며, 그 외 계정이 변경하면 MR이 차단됩니다.

---

## 🌐 Ingress

`shopping-frontend` 서비스에만 Ingress가 정의되어 있습니다.

| 항목 | 값 |
|---|---|
| IngressClass | `haproxy1` |
| Host | `shopping.project.com` |
| TLS Secret | `shopping-tls` |
| HTTP→HTTPS 리다이렉트 | HAProxy `http-response replace-value` 사용 |
