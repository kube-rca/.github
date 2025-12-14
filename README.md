# KubeRCA 프로젝트 안내

이 워크스페이스는 Kubernetes 환경에서 발생하는 알림을 수집/전달하고, UI로 확인하기 위한 구성 요소들을
디렉터리 단위(여러 Git 리포지토리)로 관리합니다.

## 구성 요소

- `PROJECT.md`: 프로젝트 배경/목표/가치/기술 스택/로드맵
- `ARCHITECTURE.md`: Alertmanager → backend → Slack 및 frontend 흐름 요약
- `../backend/`: Go 1.22 + Gin 기반 API 서버
  - Alertmanager 웹훅(`POST /webhook/alertmanager`) 수신 후 Slack으로 알림 전송
- `../frontend/`: React 18 + TypeScript + Vite + Tailwind CSS 기반 대시보드 UI
- `../helm-charts/`: Argo CD, kube-prometheus-stack, Loki, PostgreSQL 및 `kube-rca` 배포용 Helm
  차트/values
  - `../helm-charts/charts/kube-rca/README.md`: `kube-rca` 차트 문서
- `../k8s-resources/`: Argo CD Applications 및 External Secrets Operator 리소스
- `../terraform/`: Terraform Cloud 기반 인프라 코드(예: `terraform/envs/dev/`)

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

- Backend는 Slack 전송을 위해 환경변수 `SLACK_BOT_TOKEN`, `SLACK_CHANNEL_ID`를 사용합니다.
- External Secrets 관련 리소스는 `../k8s-resources/external-secrets/`에서 관리합니다.

## Git/커밋 단위

이 워크스페이스는 디렉터리 단위로 여러 Git 리포지토리를 포함합니다.
변경한 디렉터리(예: `backend/`, `frontend/`)에서 커밋을 진행합니다.
