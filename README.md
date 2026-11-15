# DevBoard

A small 3-tier project board — React, Go, Postgres — taken all the way to
production shape on AWS: infrastructure as code, GitOps delivery, a real secret
store, and distributed tracing you can actually debug with.

```
Internet ─▶ Envoy Gateway (NLB) ─┬─ /          ─▶ frontend ─▶ /api ─▶ backend ─▶ postgres
                                 └─ /api/ai/*  ─▶ ai-service ─▶ Ollama

AWS Secrets Manager ─▶ External Secrets Operator ─▶ Secret ─▶ backend + postgres

everything ──OTLP──▶ OpenTelemetry Collector ─┬─▶ Tempo      (traces)
                                              ├─▶ Loki       (logs)
                                              └─▶ Prometheus (metrics) ─▶ Grafana
```

The app itself is deliberately small. The interesting part is everything around
it — the same application deployed two ways, raw manifests and a Helm chart,
side by side, so the difference is visible rather than asserted.

---

## Run it locally

Docker is the only prerequisite. No Node, no Go, no Postgres on your machine.

```bash
make up        # build and start everything
make smoke     # quick check that all three tiers answer
make logs      # follow logs
make reset     # wipe the database and start fresh
make down      # stop
```

Frontend on <http://localhost:8080>, backend on <http://localhost:8081>.
`make setup` copies `.env.example` to `.env` on first run; ports and Postgres
credentials live there.

[`ORIGINAL-README.md`](ORIGINAL-README.md) walks the same stack up by hand —
`docker network create`, then each container individually — before handing over
to Compose. Worth doing once to see what Compose is actually doing for you.

## The services

| Service | Language | What it does |
|---------|----------|--------------|
| `frontend/` | React + Vite | The board UI; proxies `/api` to the backend; browser-side OTel tracing |
| `backend/` | Go 1.26 + Gin | REST API over Postgres, instrumented with otelgin and otelsql |
| `ai-service/` | Python + Flask | Pulls a project's tasks for context, streams the model's answer back over SSE |
| `init/postgres/` | SQL | Schema plus seed projects and tasks, loaded on first start |

The AI service is the one place a synchronous service-to-service call is
correct — it cannot answer without the task context, so that cost is paid in
the request path. It runs against a self-hosted Ollama model, so no API key and
no per-token cost.

## The mega-project: DevBoard on EKS

Fifteen chapters in [`gitops/`](gitops/README.md), each building on the last:

| Chapters | What gets built |
|----------|-----------------|
| [01–03](gitops/01-prerequisites.md) | Prerequisites, Terraform remote state bootstrap, EKS with its own VPC, subnets, NAT and IAM |
| [04–05](gitops/04-gateway-api.md) | Envoy Gateway for a public URL (Gateway API → NLB), then Argo CD |
| [06](gitops/06-secrets-with-secrets-manager.md) | AWS Secrets Manager + External Secrets Operator — no credentials in Git, no access keys anywhere |
| [07–09](gitops/07-deploy-without-helm.md) | Deploy twice: raw `k8s/` manifests, then the `helm/devboard` chart |
| [10](gitops/10-ai-feature.md) | The AI assistant, served by Ollama in-cluster |
| [11–13](gitops/11-observability.md) | OpenTelemetry instrumented three ways, ending in a chapter where you break things and find them in traces |
| [14–15](gitops/14-cicd.md) | The CI/CD loop, then teardown |

[`Deploy.md`](Deploy.md) is the command-by-command runbook version — empty AWS
account to HTTPS on your own domain, in one pass.

## Infrastructure and delivery

```
terraform/        EKS, VPC, pod identity, secrets, storage, Argo CD; bootstrap/ for remote state
k8s/              Raw manifests: deployments, services, HPA, Gateway + HTTPRoutes,
                  cert-manager Certificate, ExternalSecret, Postgres StatefulSet, RBAC
helm/devboard/    The same app as a chart
gitops/argocd/    Argo CD Applications — the app, and the platform stack
                  (cert-manager, external-secrets, Grafana, Loki, Tempo, Prometheus, Ollama)
ansible/          Provision a control box: install tools, configure AWS, clone, verify
scripts/deploy.sh One-shot deploy helper
```

## CI/CD

Twelve workflows in [`.github/workflows/`](.github/workflows/):

| Area | Workflows |
|------|-----------|
| Tests & quality | `code-tests.yml`, `code-quality.yml`, `matrix.yml`, `sonar-scan.yml` |
| Security | `devsecops.yml`, `dependency-scan.yml`, `docker-scans.yml`, `secret-scanning.yml`, `dast.yml` |
| Delivery | `docker-push.yml` (SHA-tagged images), `gitops-bump.yml` (writes the tag into the manifests and commits back for Argo CD to sync) |
| Infra | `terraform-ci.yml` (fmt, validate, tflint, Checkov) |

The GitOps loop: push to `main` → images built and tagged `sha-<short>` →
`gitops-bump` commits the new tag into the manifests → Argo CD sees the commit
and syncs the cluster. Nothing deploys by pushing to a cluster; the cluster
pulls from Git.

## Before deploying this

A few things need to be yours rather than inherited:

- **Argo CD sources** in `gitops/argocd/` point at `nitin1094/devboard`, branch `main` — already rewritten from upstream. Rename the branch and `targetRevision` needs updating too.
- **Container images** in `docker-compose.yml`, `k8s/` and `helm/` still reference the upstream `trainwithshubham/devboard-*` images. They are public and pull fine, so everything runs as-is. To build your own, set the `DOCKERHUB_USERNAME` repository variable plus the registry secret and let `docker-push.yml` populate your namespace.
- **SonarCloud** keys in `sonar-project.properties` are set to `nitin1094_devboard` / `nitin1094`; they must match the project you import in the SonarCloud UI.
- **The domain** throughout `Deploy.md` is the original author's. Substitute your own hosted zone everywhere it appears.
- **Secrets** — `devsecops.yml`, `sonar-scan.yml`, `docker-push.yml` and `terraform-ci.yml` need registry, Sonar and AWS credentials. Until those exist, those workflows fail on every push.

## Stack

React + Vite · Go 1.26 + Gin · Python + Flask · Postgres · Docker & Compose ·
Terraform · EKS · Argo CD · Helm · Gateway API (Envoy) · External Secrets +
AWS Secrets Manager · cert-manager · OpenTelemetry with Tempo, Loki, Prometheus
and Grafana · Ollama · Ansible · GitHub Actions

---

## Credits

DevBoard originates from [LondheShubham153/devboard](https://github.com/LondheShubham153/devboard)
(TrainWithShubham). The upstream repository publishes no license file, so the
original terms are whatever the author grants — this copy is kept for my own
learning and deployment practice. The original README is preserved as
[`ORIGINAL-README.md`](ORIGINAL-README.md). My changes and notes are my own.
