# Project Analysis: devops-k8s-pipeline

## 1. Overview

**Name:** devops-k8s-pipeline  
**Repository:** https://github.com/amine-hassoun/devops-k8s-pipeline  
**License:** MIT (package.json says ISC; README says MIT)

This is a **production-grade Kubernetes deployment pipeline for a Node.js REST API**. It demonstrates a complete DevOps workflow: container image build, vulnerability scanning, container registry push, Helm-based Kubernetes deployment, RBAC, NetworkPolicy, HPA, Sealed Secrets, ConfigMaps, and Prometheus/Grafana observability — all orchestrated via GitHub Actions CI/CD and verified on a live cluster.

## 2. Architecture

```
Developer --git push--> GitHub Repo --CI (build-push-deploy.yaml)--> Docker Image
                                                              |
                                                              v (trivy scan --exit-code 1)
                                                      [Scan Gate] --fail--> Pipeline Blocked
                                                      [Scan Gate] --pass--> GHCR Registry (SHA tagged)
                                                              |
                                                              v (helm upgrade --install)
                                                      Kubernetes Cluster (namespace: app)
                                                      ├── Ingress -> Service -> Pod (REST API)
                                                      ├── HPA (tiered by env, cpu target)
                                                      ├── ServiceAccount + Role + RoleBinding (least privilege)
                                                      ├── SealedSecret -> Secret (decrypted by controller)
                                                      ├── NetworkPolicy (default deny + allow from ingress-nginx)
                                                      └── ConfigMap -> Pod
                                                      └── ServiceMonitor -> Prometheus -> Grafana
```

## 3. Repository Structure

| Path | Purpose |
|------|---------|
| `src/index.js` | Node.js/Express REST API (`/health`, `/items`, `/metrics`) |
| `src/metrics.js` | prom-client instrumentation (counter + histogram + default metrics) |
| `src/index.test.js` | Jest + supertest unit tests |
| `Dockerfile` | Multi-stage build; non-root user; prunes dev deps; removes npm/npx from runtime image |
| `k8s/` | Raw Kubernetes manifests (namespace, deployment, service, configmap, hpa, ingress, networkpolicy, serviceaccount, role, rolebinding, sealedsecret) |
| `helm/app-chart/` | Helm chart with Chart.yaml, values.yaml, values-dev.yaml, values-prod.yaml, templates/ |
| `.github/workflows/build-push-deploy.yaml` | GitHub Actions CI/CD: build, Trivy scan, push to GHCR, manual Helm deploy |
| `.github/dependabot.yml` | Weekly Dependabot updates for npm and GitHub Actions |

## 4. Tech Stack

Node.js · Express · Docker · Kubernetes · Helm · Sealed Secrets · RBAC · NetworkPolicy · HPA · Prometheus · Grafana · prom-client · GitHub Actions · Trivy · Dependabot · minikube

## 5. Application Endpoints (Node.js REST API)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check, returns `{ status: "ok" }` |
| `/items` | GET | Returns the in-memory items array |
| `/items` | POST | Adds an item to the in-memory array, returns 201 |
| `/metrics` | GET | Prometheus-formatted metrics (scrapable by Prometheus via ServiceMonitor) |

Server listens on `PORT` env var (default `3000`).

## 6. Security Review — Is This Code Safe?

I reviewed all source files for malicious or harmful patterns. **Result: No viruses, no harmful code, no suspicious behavior found.**

### Checks performed

| Risk Category | Finding |
|---------------|---------|
| Network exfiltration | None. The app only listens locally on port 3000 and makes no outbound network calls. |
| `eval` / `exec` / `child_process` | None used anywhere in the source. |
| Obfuscated code | None. Code is clear, idiomatic, and well-documented. |
| Suspicious dependencies | Only `express`, `prom-client` (runtime) and `jest`, `supertest` (dev). All are mainstream, well-maintained packages. |
| Hardcoded secrets | None. Secrets are handled via Kubernetes SealedSecret / ConfigMap / envFrom. |
| Docker runtime risks | Runtime image runs as non-root user `app`; `npm`/`npx` removed from runtime stage; only production dependencies installed. |
| Kubernetes RBAC | Least privilege: Role only allows `get`/`list` on `configmaps`. ServiceAccount has `automountServiceAccountToken: false`. |
| NetworkPolicy | Default-deny ingress; only allows traffic from `ingress-nginx` namespace. |
| Image scanning | CI includes Trivy scan with `--exit-code 1` for HIGH/CRITICAL severities — a security best practice. |

### Notes on the SealedSecret

`k8s/sealedsecret.yaml` contains an encrypted `DB_PASSWORD` value. This is **expected and safe** — Sealed Secrets are encrypted with the cluster's Sealed Secrets controller key and can only be decrypted inside the cluster. The ciphertext itself reveals nothing.

## 7. How to Run Locally

### Prerequisites

- Node.js (v20+ recommended; v24 installed on this machine)
- npm

### Step 1 — Install dependencies

```bash
cd e:/projects/github/devops-k8s-pipeline
npm ci
```

### Step 2 — Run the test suite

```bash
npm test
```

Expected: 3 tests pass (`GET /health`, `GET /items`, `POST /items`).

### Step 3 — Start the API server

```bash
npm start
```

Expected: `Running on 3000`. The API is then available at `http://localhost:3000`.

### Step 4 — Verify endpoints

```bash
curl http://localhost:3000/health
curl http://localhost:3000/items
curl -X POST http://localhost:3000/items -H "Content-Type: application/json" -d '{"name":"test"}'
curl http://localhost:3000/metrics
```

### Step 5 — Stop the server

Press `Ctrl+C` in the terminal where `npm start` is running.

## 8. CI/CD Pipeline (GitHub Actions)

The workflow `.github/workflows/build-push-deploy.yaml`:

1. **Checkout** code
2. **Log in to GHCR** (GitHub Container Registry)
3. **Build and push** Docker image tagged with the commit SHA
4. **Trivy vulnerability scan** — fails the build on HIGH/CRITICAL findings
5. **Deploy** (manual `workflow_dispatch` only) — applies Helm chart to a cluster via kubeconfig secret

Deploy is intentionally **manual-only** because GitHub-hosted runners cannot reach an external Kubernetes cluster by default.

## 9. Conclusion

This is a legitimate, educational, and production-style DevOps project demonstrating secure Kubernetes deployment practices. It is safe to run locally, build, and test. No harmful or viral code was detected.

