# KubeRCA 프로젝트 안내

이 워크스페이스는 Kubernetes 환경에서 발생하는 알림을 수집/전달하고, UI로 확인하기 위한 구성 요소들을
디렉터리 단위(여러 Git 리포지토리)로 관리합니다.

![KubeRCA Logo](../img/Kube-RCA-Logo.png)

## 구성 요소

- `../PROJECT.md`: 프로젝트 배경/목표/가치/기술 스택/로드맵
- `../ARCHITECTURE.md`: Alertmanager/Slack/Agent/Auth 흐름 요약
- `../diagrams/`: 아키텍처 다이어그램(Mermaid 기반)
  - `../diagrams/system_context_diagram.md`
  - `../diagrams/alert_analysis_sequence_diagram.md`
  - `../diagrams/incident_analysis_sequence_diagram.md`
  - `../diagrams/login_sequence_diagram.md`
  - `../diagrams/entity_relationship_diagram.md`
- `../img/`: 로고/이미지 리소스
- `../../backend/`: Go + Gin 기반 API 서버
  - Alertmanager 웹훅(`POST /webhook/alertmanager`) 수신 후 Slack 알림 전송(스레드 처리 포함)
  - 인증/인시던트/알림/임베딩 API(`POST /api/v1/auth/*`, `GET /api/v1/incidents*`, `GET /api/v1/alerts*`, `POST /api/v1/embeddings*`)
  - 인시던트 숨김/복원(`PATCH /api/v1/incidents/:id`, `PATCH /api/v1/incidents/:id/unhide`, `GET /api/v1/incidents/hidden`)
  - OpenAPI 문서 엔드포인트: `GET /openapi.json`
- `../../agent/`: Python FastAPI 기반 분석 엔진 API
  - `POST /analyze`, `POST /summarize-incident` (Strands Agents + K8s/Prometheus 컨텍스트)
  - SESSION_DB 설정 시 Strands 세션 저장(PostgreSQL)
- `../../frontend/`: React 18 + TypeScript + Vite + Tailwind CSS 기반 대시보드 UI
  - 로그인/회원가입 + RCA/Alert 리스트/상세 + 숨김(뮤트) 인시던트 화면
- `../../helm-charts/`: Argo CD, kube-prometheus-stack, Loki, PostgreSQL, ingress-nginx 및 `kube-rca` 배포용 Helm
  차트/values
  - `../../helm-charts/charts/kube-rca/README.md`: `kube-rca` 차트 문서
- `../../k8s-resources/`: Argo CD Applications 및 External Secrets Operator 리소스
- `../../terraform/`: Terraform Cloud 기반 인프라 코드(예: `terraform/envs/dev/`)

## 로컬 개발

### Backend

```bash
cd backend
go mod tidy
go run .
```

```bash
cd backend
go test ./...
```

### Agent

```bash
cd agent
make install
make run
```

### Frontend

```bash
cd frontend
npm ci
npm run dev
```

```bash
cd frontend
npm run build
npm run lint
```

## 배포(Helm)

KubeRCA는 `helm-charts/charts/argo-applications`(App-of-Apps) 차트 기반으로 배포합니다.
즉, `argo-applications` 설치 이후에는 Argo CD가 하위 Application(`kube-rca`, `kube-prometheus-stack`,
`alloy`, `loki`, `db` 등)을 선언적으로 관리합니다.

```bash
cd helm-charts

# Argo CD
helm upgrade --install -n argocd argocd charts/argo-cd \
  -f charts/argo-cd/kube-rca-values.yaml

# App-of-Apps (kube-rca 및 관측성 스택/DB 포함)
helm upgrade --install -n argocd argo-applications charts/argo-applications \
  -f charts/argo-applications/kube-rca-values.yaml
```

## 설정(시크릿/환경변수)

- Backend Slack 연동: `SLACK_BOT_TOKEN`, `SLACK_CHANNEL_ID`
- Backend 인증: `JWT_SECRET`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `ALLOW_SIGNUP`, `AUTH_COOKIE_*`
- Backend 임베딩: `AI_API_KEY`
- Agent Gemini: `GEMINI_API_KEY`
- Agent 세션 저장(옵션): `SESSION_DB_HOST`, `SESSION_DB_PORT`, `SESSION_DB_NAME`, `SESSION_DB_USER`, `SESSION_DB_PASSWORD`
- External Secrets 관련 리소스는 `../../k8s-resources/external-secrets/`에서 관리합니다.

## Git/커밋 단위

이 워크스페이스는 디렉터리 단위로 여러 Git 리포지토리를 포함합니다.
변경한 디렉터리(예: `backend/`, `frontend/`)에서 커밋을 진행합니다.
