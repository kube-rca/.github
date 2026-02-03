<p align="center">
  <img src="https://raw.githubusercontent.com/kube-rca/.github/main/img/Kube-RCA-Logo-NoBG.png" alt="KubeRCA Logo" width="200"/>
</p>

<h1 align="center">KubeRCA</h1>

<p align="center">
  <strong>AI-Powered Kubernetes Incident Analysis & Root Cause Analysis Tool</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Go-1.24-00ADD8?style=flat-square&logo=go" alt="Go">
  <img src="https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/React-18-61DAFB?style=flat-square&logo=react&logoColor=black" alt="React">
  <img src="https://img.shields.io/badge/Helm-3-0F1689?style=flat-square&logo=helm" alt="Helm">
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License">
</p>

---

## Overview

KubeRCA is an open-source tool that automatically collects incident context from Kubernetes environments and provides **Root Cause Analysis (RCA)** and response guides using LLM.

When alerts fire in your cluster, KubeRCA:

1. Receives alerts via Alertmanager webhook
2. Collects relevant logs, metrics, and Kubernetes events
3. Analyzes the context using AI (Strands Agents: Gemini/OpenAI/Anthropic)
4. Provides RCA summaries and recommended actions
5. Searches similar past incidents for reference

---

## Features

- **Automated Context Collection** - Gather logs, metrics, and K8s events when alerts fire
- **AI-Powered Analysis** - LLM-based root cause analysis with Strands Agents (Gemini/OpenAI/Anthropic)
- **Similar Incident Search** - Vector similarity search using pgvector
- **Slack Integration** - Real-time notifications with threaded analysis results
- **Web Dashboard** - React-based UI for incident management
- **Helm Deployment** - Easy installation via Helm charts

---

## Architecture

```mermaid
flowchart LR
  %% External
  AM[Alertmanager]
  SL[Slack]
  LLM[LLM API<br/>(Gemini/OpenAI/Anthropic)]
  PR[Prometheus]
  K8S[Kubernetes API]

  %% Internal
  subgraph KubeRCA
    FE[Frontend<br/>React + TypeScript]
    BE[Backend<br/>Go + Gin]
    AG[Agent<br/>Python + FastAPI]
    PG[(PostgreSQL<br/>+ pgvector)]
  end

  AM -->|Webhook| BE
  BE -->|Notification| SL
  FE -->|REST API| BE
  BE -->|Analyze Request| AG
  AG -->|K8s Context| K8S
  AG -->|Metrics Query| PR
  AG -->|LLM Analysis| LLM
  BE -->|Embeddings| LLM
  BE <-->|Data| PG
  AG -.->|Session| PG
```

### Component Flow

| Step | Description                                               |
| ---- | --------------------------------------------------------- |
| 1    | Alertmanager sends alerts to Backend via webhook          |
| 2    | Backend creates/updates Incident and stores Alert         |
| 3    | Backend sends Slack notification (with thread tracking)   |
| 4    | Backend requests analysis from Agent (async)              |
| 5    | Agent collects K8s/Prometheus context                     |
| 6    | Agent performs LLM analysis via Strands Agents            |
| 7    | Backend stores analysis results and sends to Slack thread |
| 8    | Frontend displays incidents with similar incident search  |

---

## Tech Stack

### Application

| Component    | Technology                                  |
| ------------ | ------------------------------------------- |
| **Backend**  | Go 1.24 + Gin                               |
| **Agent**    | Python 3.10+ + FastAPI + Strands Agents     |
| **Frontend** | React 18 + TypeScript + Vite + Tailwind CSS |
| **Database** | PostgreSQL + pgvector                       |

### Infrastructure & Observability

| Category       | Technology                        |
| -------------- | --------------------------------- |
| **Deployment** | Helm, ArgoCD                      |
| **IaC**        | Terraform                         |
| **Monitoring** | Prometheus, Alertmanager, Grafana |
| **Logging**    | Loki, Grafana Alloy               |
| **AI/LLM**     | Strands Agents (Gemini/OpenAI/Anthropic) |

### Testing

| Category              | Technology |
| --------------------- | ---------- |
| **Chaos Engineering** | Chaos Mesh |
| **Load Testing**      | k6         |

---

## Quick Start

### Prerequisites

- Kubernetes cluster (1.25+)
- Helm 3.x
- AI provider API key (Gemini / OpenAI / Anthropic)
- PostgreSQL with pgvector extension (bundled subchart or external)
- Slack bot token + channel ID (optional)

### Installation via Helm (OCI, Public ECR)

```bash
# Optional: login to Public ECR (if your environment requires it)
aws ecr-public get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin public.ecr.aws

# Install/upgrade (chart version from charts/kube-rca/Chart.yaml)
helm upgrade --install kube-rca oci://public.ecr.aws/r5b7j2e4/kube-rca-ecr/kube-rca \
  --namespace kube-rca --create-namespace \
  --version 0.3.0 \
  -f values.yaml
```

### Installation from Source (Local Chart)

```bash
git clone https://github.com/your-org/kube-rca.git
cd kube-rca/helm-charts/main

helm upgrade --install kube-rca charts/kube-rca \
  --namespace kube-rca --create-namespace \
  -f values.yaml
```

### values.yaml Example (Gemini)

```yaml
backend:
  embedding:
    provider: "gemini"
    apiKey:
      existingSecret: "kube-rca-ai"
      key: "ai-studio-api-key"
  postgresql:
    secret:
      existingSecret: "postgresql"
      key: "password"
  slack:
    enabled: true
    secret:
      existingSecret: "kube-rca-slack"

agent:
  aiProvider: "gemini"
  gemini:
    secret:
      existingSecret: "kube-rca-ai"
      key: "ai-studio-api-key"
  prometheus:
    url: "http://prometheus-server.monitoring:9090"

frontend:
  ingress:
    enabled: true
    hosts:
      - kube-rca.example.com
```

> For OpenAI/Anthropic, set `agent.aiProvider` to `openai` or `anthropic` and point `agent.openai.secret` / `agent.anthropic.secret` to the corresponding secret key (`openai-api-key` / `anthropic-api-key`).

### Configure Alertmanager

Add the KubeRCA webhook receiver to your Alertmanager configuration:

```yaml
receivers:
  - name: "kube-rca"
    webhook_configs:
      - url: "http://<release>-backend.<namespace>.svc.cluster.local:8080/webhook/alertmanager"
        send_resolved: true

route:
  receiver: "kube-rca"
  # or add as a child route
```

Example (release: `kube-rca`, namespace: `kube-rca`):
`http://kube-rca-backend.kube-rca.svc.cluster.local:8080/webhook/alertmanager`

---

## Configuration

### Secrets (Default Names)

| Secret | Keys | Notes |
| --- | --- | --- |
| `postgresql` | `postgres-password`, `password` | PostgreSQL (Bitnami subchart) |
| `kube-rca-ai` | `ai-studio-api-key` / `openai-api-key` / `anthropic-api-key` | Keys depend on `agent.aiProvider` / `backend.embedding.provider` |
| `kube-rca-slack` | `kube-rca-slack-token`, `kube-rca-slack-channel-id` | Required if Slack enabled |

For full configuration options, see the Helm chart values at `helm-charts/main/charts/kube-rca/README.md`.

---

## Local Development

### Backend (Go)

```bash
cd backend
go mod tidy
go run .
# or
go test ./...
```

### Agent (Python)

```bash
cd agent
make install   # uv sync
make lint      # ruff check
make test      # pytest
make run       # uvicorn dev server
```

### Frontend (React)

```bash
cd frontend
npm ci
npm run dev    # development server
npm run build  # production build
npm run lint   # eslint
```

---

## Documentation

- [Architecture Details](../ARCHITECTURE.md)
- [Project Background](../PROJECT.md)
- [Helm Chart Values](../../helm-charts/charts/kube-rca/README.md)
- [Sequence Diagrams](../diagrams/)

---

## Contributing

Contributions are welcome! Please read our contributing guidelines before submitting PRs.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'feat: add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## License

This project is licensed under the MIT License - see the [LICENSE](../LICENSE) file for details.

---

<p align="center">
  Made with dedication for the Kubernetes community
</p>
