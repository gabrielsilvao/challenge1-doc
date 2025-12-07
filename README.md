# Case Técnico - Site Reliability Engineer (SRE)

## Sumário Executivo

Este documento descreve a arquitetura, decisões técnicas e implementação de uma plataforma modernizada na AWS, utilizando Kubernetes (EKS) como orquestrador de contêineres. O projeto foi desenvolvido com foco em **resiliência**, **confiabilidade** e **observabilidade**, implementando práticas de **GitOps** para deploy automatizado e seguro.

---

## 1. Visão Geral da Arquitetura

### 1.1 Fluxo de Deployment

![Developer Workflow](./img/gitops.svg)

### 1.2 Fluxo GitOps Automatizado

![GitOps](./img/automated-deployment-flow.svg)

### 1.3 Arquitetura AWS

![AWS Architecture](./img/aws-diagram.svg)

### 1.4 Observability Stack

![Observability Stack](./img/observability.svg)

---

## 2. Stack Tecnológico

### 2.1 Infraestrutura como Código (IaC)

| Componente | Tecnologia | Versão |
|------------|------------|--------|
| **IaC** | Terraform | >= 1.0.0 |
| **Backend** | S3 | - |
| **State Lock** | DynamoDB | - |
| **Estrutura** | Módulos (VPC, EKS, ECR, Helm) | - |

### 2.2 Kubernetes e Orquestração

| Componente | Tecnologia | Versão |
|------------|------------|--------|
| **Kubernetes** | Amazon EKS | 1.33 |
| **Node Group** | Managed | t3.medium |
| **Ingress** | Kong | 2.33.0 |
| **Scaling** | Metrics Server | 3.12.0 |

### 2.3 GitOps e CI/CD

| Componente | Tecnologia | Versão |
|------------|------------|--------|
| **GitOps Engine** | ArgoCD | 5.46.0 |
| **Image Updater** | Argo Image Updater | 0.9.6 |
| **Progressive Delivery** | Argo Rollouts | 2.32.0 |
| **CI Pipeline** | GitHub Actions | - |
| **Container Registry** | Amazon ECR | - |

### 2.4 Observabilidade

| Componente | Tecnologia | Versão |
|------------|------------|--------|
| **APM/Metrics/Logs** | Datadog | Agent 3.49.0 |
| **Instrumentação** | OpenTelemetry | 1.21.0 |
| **Coleta de Logs** | Datadog Agent | DaemonSet |
| **Métricas Kong** | Prometheus Plugin | - |

---

## 3. Estrutura dos Repositórios

### 3.1 Repositório: challenge1-app

```
challenge1-app/
├── .github/
│   └── workflows/
│       └── build-push-ecr.yml   # Pipeline CI/CD
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
└── main.go                      # Aplicação principal
```

**Aplicação:**
- **Linguagem**: Go 1.21
- **Framework**: net/http (stdlib)
- **Instrumentação**: OpenTelemetry SDK 1.21.0
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
                └── values/
                    ├── argocd-values.yaml
                    ├── argocd-image-updater-values.yaml
                    ├── argo-rollouts-values.yaml
                    └── kong-values.yaml
```

### 3.3 Repositório: challenge1-k8s-manifests

```
challenge1-k8s-manifests/
├── argocd/
│   └── application.yaml         # ArgoCD Application CRD
└── charts/
    └── sample-web-app/
        ├── Chart.yaml           # Metadados do chart
        ├── values.yaml          # Valores padrão
        ├── values-prod.yaml     # Valores de produção
        └── templates/
            ├── _helpers.tpl
            ├── rollout.yaml     # Argo Rollout
            ├── service.yaml
            ├── serviceaccount.yaml
            ├── ingress.yaml     # Kong Ingress
            ├── hpa.yaml
            ├── configmap.yaml
            ├── otel-configmap.yaml
            ├── otel-collector.yaml
            ├── otel-service.yaml
            ├── kong-prometheus-plugin.yaml
            └── datadog-secret.yaml
```

---

## 4. Estratégia de Rollout Seguro

### 4.1 Canary Deployment com Argo Rollouts

```yaml
spec:
  strategy:
    canary:
      canaryService: sample-web-app-canary
      stableService: sample-web-app-stable
      steps:
        - setWeight: 20
        - pause: {duration: 30s}
        - setWeight: 40
        - pause: {duration: 30s}
        - setWeight: 60
        - pause: {duration: 30s}
        - setWeight: 80
        - pause: {duration: 30s}
```

### 4.2 Fases do Rollout

| Fase | Tráfego Canary | Duração | Ação |
|------|----------------|---------|------|
| 1 | 20% | 30s | Validação inicial |
| 2 | 40% | 30s | Aumento gradual |
| 3 | 60% | 30s | Validação de carga |
| 4 | 80% | 30s | Validação final |
| 5 | 100% | - | Promoção completa |

### 4.3 Benefícios da Estratégia

1. **Mitigação de Risco**: Apenas uma fração dos usuários é exposta a novas versões inicialmente
2. **Rollback Rápido**: Em caso de falha, o tráfego é redirecionado para a versão estável
3. **Validação Gradual**: Tempo para identificar problemas antes do impacto total
4. **Zero Downtime**: Transição suave entre versões

### 4.4 Blue-Green com Shadow Traffic e AnalysisTemplate

Uma alternativa ao Canary é a estratégia **Blue-Green com Shadow Traffic**, onde o tráfego é espelhado para a nova versão sem afetar os usuários. O **AnalysisTemplate** do Argo Rollouts permite validar automaticamente a nova versão baseado em métricas.

#### AnalysisTemplate para Validação de Erros

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-analysis
  namespace: sample-web-app
spec:
  args:
    - name: service-name
      value: sample-web-app
  metrics:
    - name: error-rate
      interval: 30s
      count: 5
      successCondition: result[0] < 0.05
      failureLimit: 3
      provider:
        datadog:
          query: |
            sum:trace.http.request.hits{service:{{args.service-name}},http.status_code:5*}.as_count() /
            sum:trace.http.request.hits{service:{{args.service-name}}}.as_count()
    - name: latency-p99
      interval: 30s
      count: 5
      successCondition: result[0] < 500
      failureLimit: 3
      provider:
        datadog:
          query: |
            p99:trace.http.request.duration{service:{{args.service-name}}}
```

#### Rollout Blue-Green com Analysis

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: sample-web-app
  namespace: sample-web-app
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: sample-web-app-active
      previewService: sample-web-app-preview
      autoPromotionEnabled: false
      prePromotionAnalysis:
        templates:
          - templateName: error-rate-analysis
        args:
          - name: service-name
            value: sample-web-app-preview
      postPromotionAnalysis:
        templates:
          - templateName: error-rate-analysis
        args:
          - name: service-name
            value: sample-web-app
      scaleDownDelaySeconds: 30
```

#### Fluxo do Blue-Green com Shadow

| Fase | Ação | Validação |
|------|------|-----------|
| 1 | Deploy da versão Preview (Green) | Pods healthy |
| 2 | Shadow traffic para Preview | Espelhamento sem impacto |
| 3 | Pre-Promotion Analysis | Error rate < 5%, Latency p99 < 500ms |
| 4 | Switch de tráfego (Blue → Green) | Promoção automática se análise passar |
| 5 | Post-Promotion Analysis | Validação final em produção |
| 6 | Scale down da versão antiga | Cleanup após 30s |

#### Configuração do Shadow Traffic (Kong)

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-mirror
  namespace: sample-web-app
config:
  mirror_target: sample-web-app-preview.sample-web-app.svc.cluster.local:80
  mirror_percentage: 100
plugin: request-transformer
```

#### Benefícios do Blue-Green com Shadow

1. **Zero Risk Testing**: Tráfego espelhado não afeta usuários reais
2. **Validação Automatizada**: AnalysisTemplate verifica métricas automaticamente
3. **Rollback Instantâneo**: Switch de serviço é atômico
4. **Análise Completa**: Pre e Post promotion analysis garantem qualidade

---

## 5. SLIs e SLOs

### 5.1 Service Level Indicators (SLIs)

| SLI | Descrição | Métrica Datadog | Meta |
|-----|-----------|-----------------|------|
| **Availability** | Percentual de requests bem-sucedidos | `sum:trace.http.request.hits{status:2xx} / sum:trace.http.request.hits` | > 99.9% |
| **Latency (p50)** | Latência mediana | `p50:trace.http.request.duration{service:sample-web-app}` | < 100ms |
| **Latency (p99)** | Latência percentil 99 | `p99:trace.http.request.duration{service:sample-web-app}` | < 500ms |
| **Error Rate** | Taxa de erros 5xx | `sum:trace.http.request.errors{service:sample-web-app}` | < 0.1% |
| **Throughput** | Requests por segundo | `sum:trace.http.request.hits{service:sample-web-app}.as_rate()` | Baseline + 20% |

### 5.2 Service Level Objectives (SLOs)

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

### 5.3 Error Budget

| SLO | Objetivo | Error Budget (30 dias) |
|-----|----------|------------------------|
| Disponibilidade | 99.9% | 43.2 minutos |
| Latência | 95% < 200ms | 5% de requests lentos |
| Erros | 0.1% | 0.1% de requests com erro |

### 5.4 Queries Datadog para SLIs

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

## 6. Observabilidade

### 6.1 Três Pilares Implementados

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

### 6.2 Dashboard Datadog

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

---

## 7. Entregáveis

### 7.1 Checklist de Tarefas

| # | Tarefa | Status |
|---|--------|--------|
| 1 | Provisionar cluster EKS com Terraform | ✅ |
| 2 | Instalar e configurar ArgoCD | ✅ |
| 3 | Criar Helm chart da aplicação | ✅ |
| 4a | Pipeline - Build da imagem | ✅ |
| 4b | Pipeline - Push no ECR | ✅ |
| 4c | Pipeline - Atualização GitOps | ✅ |
| 5 | Instrumentar com OpenTelemetry | ✅ |
| 6 | Dashboard no Datadog | ✅ |
| 7a | Diagrama de arquitetura | ✅ |
| 7b | SLIs e SLOs propostos | ✅ |
| 7c | Estratégia de rollout seguro | ✅ |

### 7.2 Extras Implementados

1. **Argo Rollouts** - Progressive delivery com canary deployment
2. **Argo Image Updater** - Atualização automática de imagens
3. **Kong Ingress Controller** - API Gateway com métricas
4. **Kong Prometheus Plugin** - Métricas HTTP detalhadas
5. **HPA (Horizontal Pod Autoscaler)** - Auto-scaling baseado em CPU
6. **Structured Logging** - Logs JSON com correlação de traces
7. **Security Scan** - Trivy vulnerability scanner no pipeline

---

## 8. Conclusão

Este projeto demonstra a implementação de uma plataforma moderna de e-commerce com foco em:

- **Resiliência**: Canary deployments, rollback automático, HPA
- **Confiabilidade**: SLIs/SLOs definidos, error budgets, health checks
- **Observabilidade**: Métricas, logs e traces correlacionados no Datadog
- **Automação**: GitOps com ArgoCD, Image Updater, GitHub Actions
- **Segurança**: Private subnets, secrets management, vulnerability scanning

---

**Autor**: Gabriel Silva  
**Data**: Dezembro 2025  
**Versão**: 1.0.0
