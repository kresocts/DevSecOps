# Finalna checklist tablica za PDF predaju

Ova tablica mapira zahtjeve projekta na implementirane artefakte u repozitoriju.

## Sažetak po fazama

| Faza / područje | Status | Glavne datoteke |
|---|---|---|
| Analiza aplikacije i servisa | gotovo | `README.md`, `docs/architecture.md` |
| Lokalna kontejnerizacija | gotovo | `api/Dockerfile`, `frontend/Dockerfile`, `worker/Dockerfile` |
| Docker/Podman Compose lokalni stack | gotovo | `compose.yaml`, `.env.example` |
| Kubernetes manifesti | gotovo | `k8s/`, `docs/deployment.md` |
| CI/CD i DevSecOps | gotovo | `.github/workflows/ci-devsecops.yml`, `docs/ci-cd.md`, `docs/security/image-scan-report.md` |
| Rolling update i rollback | gotovo | `docs/rolling-update-rollback.md` |
| Incident runbook | gotovo | `docs/runbook.md` |
| Finalna dokumentacija | gotovo | `README.md`, `docs/architecture.md`, `docs/deployment.md`, `docs/security/image-scan-report.md`, `docs/runbook.md`, `docs/final-checklist.md` |

## Detaljna mapa zahtjeva

| Zahtjev iz PDF-a | Gdje je implementiran | Datoteka/folder | Status | Napomena |
|---|---|---|---|---|
| Source kod aplikacije i svih servisa | aplikacijski servisi | `api/`, `frontend/`, `worker/` | gotovo | Node.js servisi za API, frontend i worker |
| Web frontend | aplikacijski servis | `frontend/` | gotovo | Express + statički HTML/JS |
| API servis | aplikacijski servis | `api/` | gotovo | REST API s health/readiness endpointima |
| Background worker | aplikacijski servis | `worker/` | gotovo | Redis queue consumer, upisuje u PostgreSQL |
| PostgreSQL baza | lokalno i K8s | `infra/postgres/init.sql`, `compose.yaml`, `k8s/postgres.yaml` | gotovo | Compose volume i K8s StatefulSet/PVC |
| Redis queue/cache | lokalno i K8s | `compose.yaml`, `k8s/redis.yaml` | gotovo | Redis queue za narudžbe |
| Containerfile/Dockerfile za API | kontejnerizacija | `api/Dockerfile` | gotovo | multi-stage, non-root, runtime hardening |
| Containerfile/Dockerfile za frontend | kontejnerizacija | `frontend/Dockerfile` | gotovo | multi-stage, non-root, runtime hardening |
| Containerfile/Dockerfile za worker | kontejnerizacija | `worker/Dockerfile` | gotovo | multi-stage, non-root, runtime hardening |
| Multi-stage build | kontejnerizacija | `api/Dockerfile`, `frontend/Dockerfile`, `worker/Dockerfile` | gotovo | `deps`, `dev`, `runtime` stageovi |
| Minimalna runtime slika | kontejnerizacija | Dockerfileovi | gotovo | `node:20-alpine`, produkcijski dependencyji, uklonjeni nepotrebni runtime alati |
| Non-root korisnik u containeru | kontejnerizacija | Dockerfileovi, `k8s/*.yaml` | gotovo | Docker `USER node`; K8s `runAsNonRoot` |
| Tajne nisu hardkodirane u image | konfiguracija | `.env.example`, `k8s/secret.example.yaml`, `docs/deployment.md` | gotovo | stvarni Secret se kreira naredbom, ne commita se |
| Compose datoteka za lokalni razvoj | lokalni stack | `compose.yaml` | gotovo | uključuje svih pet servisa |
| Pokretanje cijelog stacka jednom naredbom | lokalni stack | `README.md`, `compose.yaml` | gotovo | `docker compose up --build` |
| Hot reload gdje je smisleno | lokalni stack | `compose.yaml`, Dockerfile `dev` stageovi | gotovo | `nodemon`, bind mount `src` direktorija |
| Environment varijable kroz primjer | lokalna konfiguracija | `.env.example` | gotovo | uključuje DB, Redis, portove, API URL i queue name |
| PostgreSQL perzistentnost kroz volume | lokalni stack | `compose.yaml` | gotovo | `postgres_data` named volume |
| Lokalni startup i shutdown dokumentirani | dokumentacija | `README.md` | gotovo | startup, shutdown i volume cleanup |
| Lokalna health validacija | dokumentacija i endpointi | `README.md`, `api/src/server.js`, `frontend/src/server.js` | gotovo | `/healthz`, `/readyz`, `/config` |
| Osnovni workflow kupnje karte | aplikacija i dokumentacija | `README.md`, `api/`, `worker/` | gotovo | API -> Redis -> worker -> PostgreSQL |
| Kubernetes/OpenShift manifesti ili Helm | produkcijski deployment | `k8s/` | gotovo | koristi Kubernetes manifeste |
| Namespace | Kubernetes | `k8s/namespace.yaml` | gotovo | `secure-ticketing` |
| ConfigMap | Kubernetes | `k8s/configmap.yaml`, `k8s/postgres-init-configmap.yaml` | gotovo | app config i PostgreSQL init SQL |
| Secret ili dokumentiran način kreiranja | Kubernetes | `k8s/secret.example.yaml`, `docs/deployment.md` | gotovo | placeholder primjer + `kubectl create secret` naredba |
| Deployment za frontend | Kubernetes | `k8s/frontend.yaml` | gotovo | 2 replike, probe, resources, securityContext |
| Deployment za API | Kubernetes | `k8s/api.yaml` | gotovo | 2 replike, rolling strategy, probe, resources |
| Deployment za worker | Kubernetes | `k8s/worker.yaml` | gotovo | worker s exec readiness/liveness probe strategijom |
| PostgreSQL StatefulSet | Kubernetes | `k8s/postgres.yaml` | gotovo | StatefulSet + PVC + init container za permissions |
| Redis Deployment | Kubernetes | `k8s/redis.yaml` | gotovo | Redis deployment s probeovima i read-only root filesystemom |
| Service objekti | Kubernetes | `k8s/services.yaml` | gotovo | frontend, api, postgres, redis |
| Ingress / vanjski pristup | Kubernetes | `k8s/ingress.yaml` | gotovo | `ticketing.example.com`, `/` i `/api` routing |
| Readiness probe | Kubernetes | `k8s/api.yaml`, `k8s/frontend.yaml`, `k8s/worker.yaml`, `k8s/postgres.yaml`, `k8s/redis.yaml` | gotovo | HTTP/exec probe prema tipu servisa |
| Liveness probe | Kubernetes | `k8s/api.yaml`, `k8s/frontend.yaml`, `k8s/worker.yaml`, `k8s/postgres.yaml`, `k8s/redis.yaml` | gotovo | HTTP/exec probe prema tipu servisa |
| Resource requests i limits | Kubernetes | `k8s/*.yaml` | gotovo | definirani za aplikacijske i data workloadove |
| ServiceAccount | Kubernetes security | `k8s/serviceaccount.yaml` | gotovo | `ticketing-app`, `ticketing-data` |
| RBAC least privilege | Kubernetes security | `k8s/rbac.yaml` | gotovo | Role bez pravila, token automount isključen |
| NetworkPolicy | Kubernetes security | `k8s/networkpolicy.yaml` | gotovo | default deny + dopušteni tokovi |
| Skeniranje slika prije deploya | DevSecOps | `.github/workflows/ci-devsecops.yml` | gotovo | Trivy image scan prije push/publish koraka |
| Sigurnosne provjere u CI/CD toku | DevSecOps | `.github/workflows/ci-devsecops.yml` | gotovo | npm audit, Trivy config scan, Trivy image scan |
| Quality gate prije objave/deploya | DevSecOps | `.github/workflows/ci-devsecops.yml` | gotovo | HIGH/CRITICAL image nalazi ruše job |
| Build i opcionalna objava imagea u registru | CI/CD | `.github/workflows/ci-devsecops.yml`, `docs/ci-cd.md` | gotovo | GHCR push nakon quality gatea za `main`/tagove |
| Tagging imagea | CI/CD | `.github/workflows/ci-devsecops.yml`, `docs/ci-cd.md` | gotovo | branch/tag + short SHA i immutable `sha-<short-sha>` tag |
| Scan izvještaj | DevSecOps dokumentacija | `docs/security/image-scan-report.md` | gotovo | opis alata, quality gatea, nalaza, korektivnih mjera i validacije |
| SARIF artifacti | CI/CD | `.github/workflows/ci-devsecops.yml` | gotovo | 4 artifacta u GitHub Actions runu |
| Observability/troubleshooting | dokumentacija | `README.md`, `docs/architecture.md`, `docs/runbook.md` | gotovo | health endpoints, logs, rollout status, diagnostic commands |
| Rolling update | deployment operacije | `docs/rolling-update-rollback.md` | gotovo | `kubectl set image`, `rollout status`, history |
| Rollback | deployment operacije | `docs/rolling-update-rollback.md` | gotovo | `kubectl rollout undo` i validacija |
| Runbook za pad PostgreSQL baze | runbook | `docs/runbook.md` | gotovo | simptom, uzrok, dijagnostika, korekcija, validacija, rollback/restore napomena |
| Runbook za loš image tag | runbook | `docs/runbook.md` | gotovo | rollout failure, image pull, undo rollback |
| Runbook za neispravan Secret/env | runbook | `docs/runbook.md` | gotovo | Secret/config dijagnostika i korekcija |
| README za lokalno pokretanje | dokumentacija | `README.md` | gotovo | preduvjeti, startup/shutdown, health, workflow, build, test, linkovi |
| Arhitektura i međuservisna komunikacija | dokumentacija | `docs/architecture.md` | gotovo | servisne uloge, tokovi, ports, env, security, observability |
| Deployment dokumentacija | dokumentacija | `docs/deployment.md` | gotovo | K8s apply, Secret, validation, cleanup, troubleshooting |
| CI/CD dokumentacija | dokumentacija | `docs/ci-cd.md` | gotovo | workflow, tagging, publish policy, quality gate |
| Finalna checklist tablica | dokumentacija | `docs/final-checklist.md` | gotovo | ova datoteka |

## Posebna provjera iz radnog plana

| Provjera | Datoteka/folder | Status | Napomena |
|---|---|---|---|
| Dockerfile/Containerfile postoje | `api/Dockerfile`, `frontend/Dockerfile`, `worker/Dockerfile` | gotovo | tri aplikacijska servisa |
| `compose.yaml` postoji | `compose.yaml` | gotovo | lokalni stack |
| `.env.example` postoji | `.env.example` | gotovo | lokalni config template |
| PostgreSQL volume postoji | `compose.yaml` | gotovo | `postgres_data` |
| Health endpoint validacija postoji | `README.md`, `api/src/server.js`, `frontend/src/server.js` | gotovo | `/healthz`, `/readyz`, `/config` |
| Kubernetes manifesti postoje | `k8s/` | gotovo | Kustomize folder |
| ConfigMap i Secret postoje | `k8s/configmap.yaml`, `k8s/secret.example.yaml` | gotovo | Secret primjer bez stvarne lozinke |
| Readiness/liveness probe postoje | `k8s/*.yaml` | gotovo | aplikacijski i data workloadovi |
| Ingress/Route postoji | `k8s/ingress.yaml` | gotovo | Kubernetes Ingress; za OpenShift Route bi se izradio ekvivalent |
| Resource requests/limits postoje | `k8s/*.yaml` | gotovo | definirani po workloadu |
| RBAC/service account postoji | `k8s/serviceaccount.yaml`, `k8s/rbac.yaml` | gotovo | least privilege |
| NetworkPolicy postoji ili je objašnjeno | `k8s/networkpolicy.yaml`, `docs/deployment.md` | gotovo | default deny + dopušteni tokovi |
| CI/CD postoji | `.github/workflows/ci-devsecops.yml` | gotovo | GitHub Actions |
| Image scan postoji | `.github/workflows/ci-devsecops.yml`, `docs/security/image-scan-report.md` | gotovo | Trivy |
| Rolling update/rollback dokumentiran | `docs/rolling-update-rollback.md` | gotovo | naredbe i validacija |
| Runbook postoji | `docs/runbook.md` | gotovo | 3 scenarija |
| Security report postoji | `docs/security/image-scan-report.md` | gotovo | nalazi i korekcije |

## Validacijski status

| Validacija | Status | Dokaz / napomena |
|---|---|---|
| Lokalni Compose stack | gotovo | `docker compose up --build -d`, health endpointi i purchase workflow validirani |
| Kubernetes deploy | gotovo | Docker Desktop Kubernetes, svi podovi Ready, `/readyz` i purchase workflow validirani |
| Rolling update | gotovo | `ticketing-api:v2` rollout uspješno završen |
| Rollback | gotovo | rollback vratio image na `ticketing-api:local`, `/readyz` ostao `ready` |
| Trivy image scan | gotovo | API, frontend i worker: 0 HIGH/CRITICAL vulnerabilities |
| Trivy config/IaC scan | gotovo | svi Docker/K8s targets: 0 HIGH/CRITICAL misconfigurations |
| GitHub Actions run | gotovo | workflow run `test #2` završio sa `Success`, 4 artifacta |

## Preostali rizici / ograničenja

| Rizik / ograničenje | Status | Napomena |
|---|---|---|
| Nema punih automatiziranih unit/integration testova | poznato ograničenje | CI koristi minimalni gate: `npm ci`, `node -c`, `npm audit`, Trivy |
| Ingress ovisi o ingress controlleru | ok za demo | lokalno je validirano port-forwardom; Ingress treba controller |
| Lokalni `:local` image tagovi u K8s manifestima | ok za lab | u produkciji zamijeniti GHCR/registry tagovima |
| Compose izlaže PostgreSQL i Redis portove na host | ok za dev | u produkciji su interni ClusterIP servisi i NetworkPolicy |
| `.env.example` ima demo lozinku | ok za lokalni primjer | produkcija koristi Kubernetes Secret |
