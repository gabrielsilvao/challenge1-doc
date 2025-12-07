# Documentação Técnica - Plataforma E-commerce Modernizada

## Sumário Executivo

Este documento descreve a arquitetura, decisões técnicas e implementação de uma plataforma de e-commerce modernizada na AWS, utilizando Kubernetes (EKS) como orquestrador de contêineres. O projeto foi desenvolvido com foco em **resiliência**, **confiabilidade** e **observabilidade**, implementando práticas de **GitOps** para deploy automatizado e seguro.

---

## 1. Visão Geral da Arquitetura

### 1.1 Diagrama de Arquitetura

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    DEVELOPER WORKFLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐   │
│  │   Developer  │─────▶│    GitHub    │─────▶│   GitHub     │─────▶│     ECR      │   │
│  │              │ push │  Repository  │trigger│   Actions    │ push │   Registry   │   │
│  └──────────────┘      │              │      │  (CI/CD)     │      │              │   │
│                        │ challenge1-  │      └──────────────┘      └──────┬───────┘   │
│                        │ app (main)   │                                   │           │
│                        └──────────────┘                                   │           │
│                                                                           │           │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              GitOps Flow                                          │ │
│  │                                                                                   │ │
│  │  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                   │ │
│  │  │   GitHub     │◀────▶│    ArgoCD    │◀────▶│  Argo Image  │◀── watches ──────┼─┘
│  │  │  Repository  │ sync │              │      │   Updater    │    ECR for        │
│  │  │ challenge1-  │      └──────┬───────┘      └──────────────┘    new images     │
│  │  │ k8s-manifests│             │                                                  │
│  │  └──────────────┘             │ deploys                                          │
│  │                               ▼                                                  │
│  └──────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                      AWS CLOUD                                           │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                              VPC (10.0.0.0/16)                                   │   │
│  │                                                                                   │   │
│  │   ┌─────────────────────────────┐    ┌─────────────────────────────┐            │   │
│  │   │    Public Subnets          │    │    Private Subnets          │            │   │
│  │   │    (10.0.1.0/24,           │    │    (10.0.3.0/24,            │            │   │
│  │   │     10.0.2.0/24)           │    │     10.0.4.0/24)            │            │   │
│  │   │                            │    │                              │            │   │
│  │   │  ┌──────────────────────┐ │    │  ┌──────────────────────┐   │            │   │
│  │   │  │    NAT Gateway       │ │    │  │    EKS Node Group    │   │            │   │
│  │   │  │                      │ │    │  │    (t3.medium)       │   │            │   │
│  │   │  └──────────────────────┘ │    │  │                      │   │            │   │
│  │   │                            │    │  │  ┌────────────────┐ │   │            │   │
│  │   │  ┌──────────────────────┐ │    │  │  │ sample-web-app │ │   │            │   │
│  │   │  │  Kong Ingress (ALB)  │◀┼────┼──┼──│    (Go 1.21)   │ │   │            │   │
│  │   │  │  (Load Balancer)     │ │    │  │  │                │ │   │            │   │
│  │   │  └──────────────────────┘ │    │  │  └────────────────┘ │   │            │   │
│  │   │                            │    │  │                      │   │            │   │
│  │   └─────────────────────────────┘    │  │  ┌────────────────┐ │   │            │   │
│  │                                       │  │  │  ArgoCD +      │ │   │            │   │
│  │   ┌─────────────────────────────┐    │  │  │  Argo Rollouts │ │   │            │   │
│  │   │       Internet Gateway      │    │  │  │                │ │   │            │   │
│  │   └─────────────────────────────┘    │  │  └────────────────┘ │   │            │   │
│  │                                       │  │                      │   │            │   │
│  │                                       │  │  ┌────────────────┐ │   │            │   │
│  │                                       │  │  │ Datadog Agent  │ │   │            │   │
│  │                                       │  │  │ (DaemonSet)    │ │   │            │   │
│  │                                       │  │  └────────────────┘ │   │            │   │
│  │                                       │  │                      │   │            │   │
│  │                                       │  │  ┌────────────────┐ │   │            │   │
│  │                                       │  │  │ OTel Collector │ │   │            │   │
│  │                                       │  │  └────────────────┘ │   │            │   │
│  │                                       │  │                      │   │            │   │
│  │                                       │  └──────────────────────┘   │            │   │
│  │                                       │                              │            │   │
│  │                                       └─────────────────────────────┘            │   │
│  │                                                                                   │   │
│  │   ┌───────────────────┐                                                          │   │
│  │   │ EKS Control Plane │ (Managed by AWS)                                         │   │
│  │   │ (Kubernetes 1.31) │                                                          │   │
│  │   └───────────────────┘                                                          │   │
│  │                                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐              │
│  │        ECR        │    │    S3 Bucket      │    │    IAM Roles      │              │
│  │  (Container       │    │  (Terraform       │    │  (IRSA enabled)   │              │
│  │   Registry)       │    │   State)          │    │                   │              │
│  └───────────────────┘    └───────────────────┘    └───────────────────┘              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  OBSERVABILITY STACK                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                              Datadog (us5.datadoghq.com)                         │   │
│  │                                                                                   │   │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │   │
│  │   │   Metrics   │    │    Logs     │    │   Traces    │    │ Dashboards  │      │   │
│  │   │             │    │             │    │    (APM)    │    │             │      │   │
│  │   │ • CPU/Memory│    │ • App Logs  │    │ • Spans     │    │ • SLIs/SLOs │      │   │
│  │   │ • HTTP Reqs │    │ • Kong Logs │    │ • Latency   │    │ • Alerts    │      │   │
│  │   │ • Custom    │    │ • K8s Events│    │ • Errors    │    │ • Service   │      │   │
│  │   │   Metrics   │    │             │    │             │    │   Map       │      │   │
│  │   └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘      │   │
│  │                                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│                    ▲                    ▲                    ▲                          │
│                    │                    │                    │                          │
│              OTLP/gRPC            DogStatsD             Log Collection                  │
│                    │                    │                    │                          │
│  ┌─────────────────┴────────────────────┴────────────────────┴─────────────────────┐   │
│  │                         Datadog Agent (DaemonSet)                                │   │
│  │                                                                                   │   │
│  │   • OTLP Receiver (port 4317/4318)                                               │   │
│  │   • Log Collection from /var/log/pods                                            │   │
│  │   • Kubernetes Integration                                                        │   │
│  │   • APM Trace Agent                                                              │   │
│  │                                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Fluxo de Dados

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              REQUEST FLOW                                             │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   User Request                                                                       │
│        │                                                                             │
│        ▼                                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│   │   Internet  │───▶│  Kong       │───▶│  Kubernetes │───▶│ sample-web- │         │
│   │             │    │  Ingress    │    │  Service    │    │ app Pod     │         │
│   └─────────────┘    │  Controller │    │             │    │             │         │
│                      │             │    └─────────────┘    │  ┌───────┐  │         │
│                      │  • Rate     │                       │  │  Go   │  │         │
│                      │    Limiting │                       │  │ 1.21  │  │         │
│                      │  • Metrics  │                       │  └───┬───┘  │         │
│                      │  • Routing  │                       │      │      │         │
│                      └─────────────┘                       │      ▼      │         │
│                                                            │  ┌───────┐  │         │
│                                                            │  │ OTel  │  │         │
│                                                            │  │  SDK  │──┼────┐    │
│                                                            │  └───────┘  │    │    │
│                                                            └─────────────┘    │    │
│                                                                               │    │
│                      ┌────────────────────────────────────────────────────────┘    │
│                      │                                                              │
│                      ▼                                                              │
│              ┌───────────────┐                                                      │
│              │ Datadog Agent │                                                      │
│              │ OTLP Receiver │                                                      │
│              │ (4317/4318)   │                                                      │
│              └───────┬───────┘                                                      │
│                      │                                                              │
│                      ▼                                                              │
│              ┌───────────────┐                                                      │
│              │   Datadog     │                                                      │
│              │   Platform    │                                                      │
│              └───────────────┘                                                      │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Stack Tecnológico e Justificativas

### 2.1 Infraestrutura como Código (IaC)

| Componente | Tecnologia | Versão | Justificativa |
|------------|------------|--------|---------------|
| **IaC** | Terraform | >= 1.0.0 | Padrão de mercado para IaC, suporte robusto à AWS, estado remoto, módulos reutilizáveis |
| **Backend** | S3 + DynamoDB | - | Estado remoto com locking para trabalho em equipe |
| **Estrutura** | Módulos | - | Separação de responsabilidades (VPC, EKS, ECR, Helm) |

**Decisões Técnicas:**
- **Módulos separados**: Facilita manutenção, testes e reutilização
- **Backend remoto S3**: Permite colaboração e mantém histórico do estado
- **Variáveis centralizadas**: Um único arquivo `variables.tf` para configurações globais

### 2.2 Kubernetes e Orquestração

| Componente | Tecnologia | Versão | Justificativa |
|------------|------------|--------|---------------|
| **Kubernetes** | Amazon EKS | 1.31 | Managed service, integração nativa AWS, auto-scaling |
| **Node Group** | Managed | t3.medium | Custo-benefício para workloads moderados |
| **Ingress** | Kong | 2.33.0 | API Gateway nativo K8s, métricas Prometheus, rate limiting |
| **Service Mesh** | - | - | Não implementado (simplicidade, Kong já oferece observabilidade) |

**Decisões Técnicas:**
- **EKS Managed Node Groups**: Simplifica gerenciamento de nodes, atualizações automáticas
- **Private Subnets para Workers**: Segurança - nodes não expostos diretamente à internet
- **Kong sobre NGINX**: Recursos avançados de API Gateway, suporte a plugins, métricas nativas

### 2.3 GitOps e CI/CD

| Componente | Tecnologia | Versão | Justificativa |
|------------|------------|--------|---------------|
| **GitOps Engine** | ArgoCD | 5.46.0 | Padrão de mercado, UI intuitiva, sync automático |
| **Image Updater** | Argo Image Updater | 0.9.6 | Automação de atualizações de imagem |
| **Progressive Delivery** | Argo Rollouts | 2.32.0 | Canary/Blue-Green deployments nativos |
| **CI Pipeline** | GitHub Actions | - | Integração nativa GitHub, custo zero para repos públicos |
| **Container Registry** | Amazon ECR | - | Integração IAM, proximidade com EKS |

**Decisões Técnicas:**
- **ArgoCD sobre Flux**: UI rica, melhor experiência de debug, comunidade maior
- **Argo Rollouts sobre Deployment nativo**: Canary deployment com análise automatizada
- **Image Updater**: Elimina necessidade de commit manual para atualizar versões

### 2.4 Observabilidade

| Componente | Tecnologia | Versão | Justificativa |
|------------|------------|--------|---------------|
| **APM/Metrics/Logs** | Datadog | Agent 3.49.0 | Plataforma unificada, correlação automática |
| **Instrumentação** | OpenTelemetry | 1.21.0 | Vendor-neutral, padrão CNCF |
| **Coleta de Logs** | Datadog Agent | DaemonSet | Coleta automática de /var/log/pods |
| **Métricas Kong** | Prometheus Plugin | - | HTTP metrics (latência, status codes, throughput) |

**Decisões Técnicas:**
- **Datadog sobre Prometheus+Grafana**: 
  - Menor overhead operacional
  - Correlação automática entre métricas, logs e traces
  - Alerting integrado
  - APM out-of-the-box
- **OpenTelemetry sobre SDK proprietário**:
  - Vendor-neutral (migração futura simplificada)
  - Padrão CNCF
  - Suporte multi-linguagem

---

## 3. Estrutura dos Repositórios

### 3.1 Repositório: challenge1-app

```
challenge1-app/
├── .github/
│   └── workflows/
│       └── ci.yaml              # Pipeline CI/CD
├── pkg/
│   ├── middleware/
│   │   └── tracing.go           # Middleware de tracing e logging
│   └── telemetry/
│       └── telemetry.go         # Inicialização OpenTelemetry
├── templates/
│   ├── index.html               # Template da página inicial
│   └── echo.html                # Template da página echo
├── Dockerfile                   # Multi-stage build
├── go.mod                       # Dependências Go
├── go.sum                       # Lock de dependências
└── main.go                      # Aplicação principal
```

**Aplicação:**
- **Linguagem**: Go 1.21
- **Framework**: net/http (stdlib)
- **Instrumentação**: OpenTelemetry SDK
- **Endpoints**:
  - `/` - Página inicial
  - `/echo` - Página de echo com informações do request
  - `/health` - Health check (liveness)
  - `/ready` - Readiness probe

### 3.2 Repositório: challenge1-infra-tf

```
challenge1-infra-tf/
└── aws/
    └── terraform/
        ├── main.tf              # Orquestração de módulos
        ├── variables.tf         # Variáveis globais
        ├── outputs.tf           # Outputs do Terraform
        ├── providers.tf         # Configuração de providers
        ├── backend/             # Configuração do backend S3
        │   ├── main.tf
        │   ├── variables.tf
        │   └── outputs.tf
        └── modules/
            ├── vpc/             # Módulo de rede
            ├── eks/             # Módulo do cluster EKS
            ├── ecr/             # Módulo do registry
            └── helm/            # Módulo de Helm charts
                └── values/      # Arquivos de values
                    ├── argocd-values.yaml
                    ├── argocd-image-updater-values.yaml
                    ├── argo-rollouts-values.yaml
                    ├── kong-values.yaml
                    └── datadog-values.yaml
```

### 3.3 Repositório: challenge1-k8s-manifests

```
challenge1-k8s-manifests/
├── argocd/
│   └── application.yaml         # ArgoCD Application CRD
├── charts/
│   └── sample-web-app/
│       ├── Chart.yaml           # Metadados do chart
│       ├── values.yaml          # Valores padrão
│       ├── values-prod.yaml     # Valores de produção
│       └── templates/
│           ├── _helpers.tpl     # Template helpers
│           ├── rollout.yaml     # Argo Rollout (substitui Deployment)
│           ├── service.yaml     # Kubernetes Service
│           ├── serviceaccount.yaml
│           ├── ingress.yaml     # Kong Ingress
│           ├── hpa.yaml         # Horizontal Pod Autoscaler
│           ├── configmap.yaml   # Configurações
│           ├── otel-configmap.yaml
│           ├── otel-collector.yaml
│           ├── otel-service.yaml
│           ├── kong-prometheus-plugin.yaml
│           └── datadog-secret.yaml
└── docs/
    └── TECHNICAL_DOCUMENTATION.md
```

---

## 4. Pipeline CI/CD

### 4.1 GitHub Actions Workflow

```yaml
# .github/workflows/ci.yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and Push
        env:
          ECR_REGISTRY: 241726813110.dkr.ecr.us-east-1.amazonaws.com
          IMAGE_NAME: sample-web-app
        run: |
          docker build -t $ECR_REGISTRY/$IMAGE_NAME:${{ github.sha }} \
                       -t $ECR_REGISTRY/$IMAGE_NAME:latest \
                       -t $ECR_REGISTRY/$IMAGE_NAME:main .
          docker push $ECR_REGISTRY/$IMAGE_NAME --all-tags
```

### 4.2 Fluxo de Deploy Automatizado

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           AUTOMATED DEPLOYMENT FLOW                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   1. Developer Push                                                                 │
│      │                                                                              │
│      ▼                                                                              │
│   ┌─────────────────┐                                                              │
│   │  GitHub Actions │ ─── Build & Test ───▶ Push to ECR                            │
│   └─────────────────┘                              │                                │
│                                                    │                                │
│   2. Image Available                               ▼                                │
│      │                              ┌─────────────────────────┐                    │
│      │                              │  ECR Repository         │                    │
│      │                              │  sample-web-app:main    │                    │
│      │                              └───────────┬─────────────┘                    │
│      │                                          │                                   │
│   3. Auto Detection                             │ watches                           │
│      │                              ┌───────────▼─────────────┐                    │
│      │                              │  Argo Image Updater     │                    │
│      │                              │  (every 2 minutes)      │                    │
│      │                              └───────────┬─────────────┘                    │
│      │                                          │                                   │
│   4. Application Update                         │ updates                           │
│      │                              ┌───────────▼─────────────┐                    │
│      │                              │  ArgoCD Application     │                    │
│      │                              │  spec.source.helm.      │                    │
│      │                              │  parameters[image.tag]  │                    │
│      │                              └───────────┬─────────────┘                    │
│      │                                          │                                   │
│   5. Sync & Rollout                             │ syncs                             │
│      │                              ┌───────────▼─────────────┐                    │
│      │                              │  Argo Rollouts          │                    │
│      │                              │  (Canary Strategy)      │                    │
│      │                              └───────────┬─────────────┘                    │
│      │                                          │                                   │
│   6. Progressive Delivery                       ▼                                   │
│      │                              ┌─────────────────────────┐                    │
│      │                              │  Step 1: 20% traffic    │                    │
│      │                              │  Step 2: 50% traffic    │                    │
│      │                              │  Step 3: 100% traffic   │                    │
│      │                              │  (with pause between)   │                    │
│      │                              └─────────────────────────┘                    │
│      │                                                                              │
│   7. Completed ✓                                                                    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Estratégia de Rollout Seguro

### 5.1 Canary Deployment com Argo Rollouts

A estratégia de rollout implementada utiliza **Canary Deployment** para minimizar riscos e permitir rollback automático em caso de problemas.

```yaml
# Configuração do Argo Rollout
spec:
  strategy:
    canary:
      canaryService: sample-web-app-canary
      stableService: sample-web-app-stable
      steps:
        # Step 1: 20% do tráfego para canary
        - setWeight: 20
        - pause: { duration: 2m }
        
        # Step 2: 50% do tráfego
        - setWeight: 50
        - pause: { duration: 2m }
        
        # Step 3: 80% do tráfego
        - setWeight: 80
        - pause: { duration: 2m }
        
        # Step 4: 100% - promoção completa
        - setWeight: 100
```

### 5.2 Fases do Rollout

| Fase | Tráfego Canary | Duração | Ação |
|------|----------------|---------|------|
| 1 | 20% | 2 min | Validação inicial, métricas básicas |
| 2 | 50% | 2 min | Validação de carga, latência |
| 3 | 80% | 2 min | Validação final antes da promoção |
| 4 | 100% | - | Promoção completa, pods antigos removidos |

### 5.3 Critérios de Rollback Automático

```yaml
analysis:
  templates:
    - templateName: success-rate
  args:
    - name: service-name
      value: sample-web-app
      
# AnalysisTemplate
spec:
  metrics:
    - name: success-rate
      successCondition: result[0] >= 0.95  # 95% success rate
      provider:
        datadog:
          query: |
            sum:trace.http.request.hits{service:sample-web-app,status:2xx} / 
            sum:trace.http.request.hits{service:sample-web-app}
```

### 5.4 Benefícios da Estratégia

1. **Mitigação de Risco**: Apenas uma fração dos usuários é exposta a novas versões inicialmente
2. **Rollback Rápido**: Em caso de falha, o tráfego é imediatamente redirecionado para a versão estável
3. **Validação Gradual**: Tempo para identificar problemas antes do impacto total
4. **Zero Downtime**: Transição suave entre versões

---

## 6. SLIs e SLOs

### 6.1 Service Level Indicators (SLIs)

| SLI | Descrição | Métrica Datadog | Meta |
|-----|-----------|-----------------|------|
| **Availability** | Percentual de requests bem-sucedidos | `sum:trace.http.request.hits{status:2xx} / sum:trace.http.request.hits` | > 99.9% |
| **Latency (p50)** | Latência mediana | `p50:trace.http.request.duration{service:sample-web-app}` | < 100ms |
| **Latency (p99)** | Latência percentil 99 | `p99:trace.http.request.duration{service:sample-web-app}` | < 500ms |
| **Error Rate** | Taxa de erros 5xx | `sum:trace.http.request.errors{service:sample-web-app}` | < 0.1% |
| **Throughput** | Requests por segundo | `sum:trace.http.request.hits{service:sample-web-app}.as_rate()` | Baseline + 20% |

### 6.2 Service Level Objectives (SLOs)

#### SLO 1: Disponibilidade
```
Objetivo: 99.9% de disponibilidade mensal
Window: 30 dias rolling
Budget: 43.2 minutos de downtime/mês
Cálculo: (Requests 2xx + 3xx) / Total Requests * 100
```

#### SLO 2: Latência
```
Objetivo: 95% dos requests < 200ms
Window: 7 dias rolling
Cálculo: Percentual de requests com latência < 200ms
```

#### SLO 3: Taxa de Erros
```
Objetivo: < 0.1% de erros 5xx
Window: 24 horas rolling
Cálculo: Requests 5xx / Total Requests * 100
```

### 6.3 Error Budget

| SLO | Objetivo | Error Budget (30 dias) |
|-----|----------|------------------------|
| Disponibilidade | 99.9% | 43.2 minutos |
| Latência | 95% < 200ms | 5% de requests lentos |
| Erros | 0.1% | 0.1% de requests com erro |

### 6.4 Queries Datadog para SLIs

```sql
-- Availability SLI
sum:trace.http.request.hits{service:sample-web-app,http.status_code:2*}.as_count() / 
sum:trace.http.request.hits{service:sample-web-app}.as_count() * 100

-- Latency P99
p99:trace.http.request.duration{service:sample-web-app}

-- Error Rate
sum:trace.http.request.hits{service:sample-web-app,http.status_code:5*}.as_count() / 
sum:trace.http.request.hits{service:sample-web-app}.as_count() * 100

-- Request Rate
sum:trace.http.request.hits{service:sample-web-app}.as_rate()
```

---

## 7. Observabilidade

### 7.1 Três Pilares Implementados

#### Métricas
- **Fonte**: OpenTelemetry SDK + Datadog Agent
- **Tipos**:
  - Application metrics (request count, latency)
  - Infrastructure metrics (CPU, memory, network)
  - Custom business metrics
- **Endpoint**: OTLP → Datadog Agent (4318) → Datadog Platform

#### Logs
- **Coleta**: Datadog Agent DaemonSet
- **Formato**: JSON estruturado
- **Campos**:
  ```json
  {
    "timestamp": "2025-12-07T12:00:00Z",
    "level": "info",
    "service": "sample-web-app",
    "trace_id": "abc123...",
    "span_id": "def456...",
    "http.method": "GET",
    "http.url": "/",
    "http.status_code": 200,
    "http.duration_ms": 12.5
  }
  ```
- **Correlação**: trace_id e span_id para correlação com traces

#### Traces
- **Instrumentação**: OpenTelemetry Go SDK
- **Propagação**: W3C Trace Context
- **Spans**:
  - HTTP request span (root)
  - Handler span
  - Template rendering span
- **Atributos**:
  - `http.method`, `http.url`, `http.status_code`
  - `http.host`, `http.user_agent`
  - Custom attributes

### 7.2 Instrumentação da Aplicação

```go
// Middleware de Tracing
func TracingMiddleware(tel *telemetry.Telemetry, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Create span
        ctx, span := tel.Tracer.Start(r.Context(), r.URL.Path,
            trace.WithSpanKind(trace.SpanKindServer),
            trace.WithAttributes(
                attribute.String("http.method", r.Method),
                attribute.String("http.url", r.URL.String()),
            ),
        )
        defer span.End()
        
        // Execute handler
        rw := newResponseWriter(w)
        next.ServeHTTP(rw, r.WithContext(ctx))
        
        // Record metrics and attributes
        duration := time.Since(start)
        span.SetAttributes(
            attribute.Int("http.status_code", rw.statusCode),
            attribute.Float64("http.duration_ms", float64(duration.Milliseconds())),
        )
        
        // Structured JSON log
        log.Printf(`{"timestamp":"%s","level":"info","trace_id":"%s",...}`, ...)
        
        // Record metrics
        tel.RecordRequest(ctx, r.Method, r.URL.Path, rw.statusCode, duration)
    })
}
```

### 7.3 Dashboard Datadog

O dashboard criado inclui:

1. **Overview Panel**
   - Request rate (requests/sec)
   - Error rate (%)
   - Latency P50/P95/P99

2. **Infrastructure Panel**
   - Pod CPU/Memory usage
   - Node health
   - Container restarts

3. **APM Panel**
   - Service map
   - Trace waterfall
   - Error tracking

4. **Business Panel**
   - Endpoint breakdown
   - Geographic distribution
   - User experience metrics

---

## 8. Segurança

### 8.1 Medidas Implementadas

| Camada | Medida | Descrição |
|--------|--------|-----------|
| **Rede** | Private Subnets | Worker nodes em subnets privadas |
| **Rede** | Security Groups | Regras mínimas necessárias |
| **IAM** | IRSA | Service accounts com roles IAM específicas |
| **Secrets** | Kubernetes Secrets | Datadog API key em Secret |
| **Container** | Non-root user | Aplicação roda como non-root |
| **Registry** | ECR Scanning | Scan de vulnerabilidades automático |
| **GitOps** | SSH Keys | Autenticação SSH para repos Git |

### 8.2 Boas Práticas Aplicadas

1. **Princípio do Menor Privilégio**
   - IAM roles com permissões mínimas
   - Service accounts dedicadas por aplicação

2. **Defesa em Profundidade**
   - Múltiplas camadas de segurança
   - Network policies (quando aplicável)

3. **Secrets Management**
   - Secrets nunca em código
   - Rotação via External Secrets (futuro)

4. **Image Security**
   - Multi-stage build (imagem mínima)
   - Base image alpine/distroless
   - Scan de vulnerabilidades

---

## 9. Componentes Instalados via Helm

| Release | Chart | Namespace | Propósito |
|---------|-------|-----------|-----------|
| argocd | argo/argo-cd | argocd | GitOps engine |
| argo-rollouts | argo/argo-rollouts | argo-rollouts | Progressive delivery |
| argocd-image-updater | argo/argocd-image-updater | argocd | Auto image updates |
| kong | kong/kong | kong | Ingress controller |
| metrics-server | metrics-server/metrics-server | kube-system | HPA metrics |
| datadog | datadog/datadog | datadog | Observability |

---

## 10. Configurações de Ambiente

### 10.1 Variáveis de Ambiente (Aplicação)

| Variável | Valor | Descrição |
|----------|-------|-----------|
| `PORT` | 8080 | Porta da aplicação |
| `OTEL_SERVICE_NAME` | sample-web-app | Nome do serviço para traces |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | datadog.datadog.svc.cluster.local:4318 | Endpoint OTLP |
| `DD_ENV` | production | Ambiente Datadog |
| `DD_SERVICE` | sample-web-app | Serviço Datadog |
| `DD_VERSION` | 1.0.0 | Versão da aplicação |

### 10.2 Resources (Kubernetes)

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 10.3 Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

---

## 11. Comandos Úteis

### 11.1 Terraform

```bash
# Inicializar backend
cd challenge1-infra-tf/aws/terraform/backend
terraform init && terraform apply

# Aplicar infraestrutura
cd ../
terraform init && terraform apply

# Destruir infraestrutura
terraform destroy
```

### 11.2 Kubernetes

```bash
# Configurar kubeconfig
aws eks update-kubeconfig --name eks-cluster --region us-east-1

# Verificar pods
kubectl get pods -n sample-web-app

# Verificar rollout status
kubectl argo rollouts get rollout sample-web-app -n sample-web-app

# Promover canary
kubectl argo rollouts promote sample-web-app -n sample-web-app

# Rollback
kubectl argo rollouts abort sample-web-app -n sample-web-app
```

### 11.3 ArgoCD

```bash
# Obter senha inicial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward para UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Sync manual
argocd app sync sample-web-app
```

---

## 12. URLs de Acesso

| Componente | URL | Credenciais |
|------------|-----|-------------|
| **Aplicação** | http://a43fa4c0cb2054cf5af1511ff9fc8064-bc4625f79fea8e89.elb.us-east-1.amazonaws.com | N/A |
| **ArgoCD UI** | http://a9297beb7a67a45ae9860e4c733409f5-1413116813.us-east-1.elb.amazonaws.com | admin / (kubectl get secret) |
| **Datadog** | https://us5.datadoghq.com | Credenciais da conta |
| **ECR** | 241726813110.dkr.ecr.us-east-1.amazonaws.com/sample-web-app | AWS IAM |

---

## 13. Entregáveis

### 13.1 Checklist de Tarefas

| # | Tarefa | Status | Evidência |
|---|--------|--------|-----------|
| 1 | Provisionar cluster EKS com Terraform | ✅ Completo | `terraform apply` + EKS rodando |
| 2 | Instalar e configurar ArgoCD | ✅ Completo | ArgoCD UI acessível |
| 3 | Criar Helm chart da aplicação | ✅ Completo | `charts/sample-web-app/` |
| 4a | Pipeline - Build da imagem | ✅ Completo | GitHub Actions workflow |
| 4b | Pipeline - Push no ECR | ✅ Completo | Imagem no ECR |
| 4c | Pipeline - Atualização GitOps | ✅ Completo | Argo Image Updater |
| 5 | Instrumentar com OpenTelemetry | ✅ Completo | Traces no Datadog |
| 6 | Dashboard no Datadog | ✅ Completo | Dashboard criado |
| 7a | Diagrama de arquitetura | ✅ Completo | Este documento |
| 7b | SLIs e SLOs propostos | ✅ Completo | Seção 6 deste documento |
| 7c | Estratégia de rollout seguro | ✅ Completo | Seção 5 deste documento |

### 13.2 Extras Implementados

Além dos requisitos, foram implementados:

1. **Argo Rollouts** - Progressive delivery com canary deployment
2. **Argo Image Updater** - Atualização automática de imagens
3. **Kong Ingress Controller** - API Gateway com métricas
4. **Kong Prometheus Plugin** - Métricas HTTP detalhadas
5. **HPA (Horizontal Pod Autoscaler)** - Auto-scaling baseado em CPU
6. **Structured Logging** - Logs JSON com correlação de traces
7. **Multi-stage Dockerfile** - Imagem otimizada

---

## 14. Próximos Passos (Roadmap)

### Curto Prazo (1-2 sprints)
- [ ] Implementar Network Policies
- [ ] Adicionar External Secrets para gestão de secrets
- [ ] Configurar alertas baseados em SLOs

### Médio Prazo (1-2 meses)
- [ ] Implementar análise automatizada no canary (AnalysisTemplate)
- [ ] Adicionar Service Mesh (Istio/Linkerd)
- [ ] Implementar chaos engineering (Chaos Monkey)

### Longo Prazo (3-6 meses)
- [ ] Multi-cluster deployment
- [ ] Disaster recovery automatizado
- [ ] Cost optimization com Spot instances

---

## 15. Conclusão

Este projeto demonstra a implementação de uma plataforma moderna de e-commerce com foco em:

- **Resiliência**: Canary deployments, rollback automático, HPA
- **Confiabilidade**: SLIs/SLOs definidos, error budgets, health checks
- **Observabilidade**: Métricas, logs e traces correlacionados no Datadog
- **Automação**: GitOps com ArgoCD, Image Updater, GitHub Actions
- **Segurança**: Private subnets, IRSA, secrets management

A arquitetura proposta segue as melhores práticas da indústria e padrões CNCF, permitindo escalabilidade futura e manutenção simplificada.

---

**Autor**: Gabriel Silva  
**Data**: Dezembro 2025  
**Versão**: 1.0.0
