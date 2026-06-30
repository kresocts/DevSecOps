# Secure Event Ticketing Platform

Projekt demonstrira lokalni razvoj u kontejnerima, sigurnu izradu i skeniranje container imagea, CI/CD, Kubernetes deployment, rolling update/rollback i troubleshooting dokumentaciju.

## 1. Kratki sažetak

Projekt implementira višeslojnu aplikaciju sa sljedećim servisima:

| Servis     | Uloga                                                      | Lokacija                                                       |
|------------|------------------------------------------------------------|----------------------------------------------------------------|
| `frontend` | Web sučelje za pregled evenata i kupnju karata             | `frontend/`                                                    |
| `api`      | REST API, health/readiness endpointi i zaprimanje narudžbi | `api/`                                                         |
| `worker`   | Pozadinska obrada narudžbi iz Redis queuea                 | `worker/`                                                      |   
| `postgres` | Trajna pohrana narudžbi                                    | `infra/postgres/init.sql`, `compose.yaml`, `k8s/postgres.yaml` |
| `redis`    | Queue/cache sloj                                           | `compose.yaml`, `k8s/redis.yaml`                               |
|------------|------------------------------------------------------------|----------------------------------------------------------------|


Osnovni tok aplikacije:

```text
Browser -> frontend -> API -> Redis queue -> worker -> PostgreSQL
```

Detaljna arhitektura, međuservisna komunikacija i usporedba kontejnera s virtualnim mašinama nalaze se u:

```text
docs/architecture.md
```

## 2. Struktura repozitorija

```text
.
├── api/                         # REST API servis
├── frontend/                    # Web frontend servis
├── worker/                      # Background worker servis
├── infra/postgres/init.sql      # Inicijalizacija PostgreSQL baze
├── compose.yaml                 # Lokalni Docker/Podman Compose stack
├── .env.example                 # Primjer lokalnih environment varijabli
├── k8s/                         # Kubernetes manifesti
├── .github/workflows/           # GitHub Actions CI/CD workflow
└── docs/                        # Arhitektura, runbook
```

Najvažniji dokumenti za pregled:

| Dokument | Namjena |
|--------------------------------------|---------------------------------------------------------|
| `README.md`                          | Detaljne upute za lokalno pokretanje i validaciju       |
| `docs/architecture.md`               | Arhitektura, servisi, komunikacija, kontejneri vs VM.   |
| `docs/security/image-scan-report.md` | Sigurnosni scan imagea i konfiguracije                  |
| `docs/runbook.md`                    | Incident runbook za troubleshooting                     |
|--------------------------------------|---------------------------------------------------------|


## 3. Lokalno pokretanje kroz Compose

Preduvjeti:

- Docker Desktop ili Docker Engine s Compose pluginom
- Alternativno Podman s Compose kompatibilnim alatom

Pokretanje cijelog lokalnog stacka:

```bash
cp .env.example .env
docker compose up --build -d
```

Provjera statusa:

```bash
docker compose ps
```

Health/readiness validacija:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:3000/healthz
curl http://localhost:3000/config
```

Očekivani API readiness odgovor:

```json
{"status":"ready"}
```

Osnovni workflow kupnje karte:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'

curl http://localhost:8080/tickets/orders
```

Zaustavljanje stacka:

```bash
docker compose down
```

Brisanje lokalnog PostgreSQL volumea:

```bash
docker compose down -v
```

## 4. Container imagei

Aplikacijski servisi imaju vlastite Dockerfileove:

```text
api/Dockerfile
frontend/Dockerfile
worker/Dockerfile
```

Implementirane sigurnosne prakse:

- multi-stage build,
- minimalna runtime slika na temelju `node:20-alpine`,
- production dependencyji,
- non-root korisnik `node`,
- `.dockerignore` datoteke,
- tajne se ne zapisuju u image.

Ručni build imagea:

```bash
docker build -t ticketing-api:local ./api
docker build -t ticketing-frontend:local ./frontend
docker build -t ticketing-worker:local ./worker
```

Provjera non-root korisnika:

```bash
docker run --rm ticketing-api:local id
docker run --rm ticketing-frontend:local id
docker run --rm ticketing-worker:local id
```

Očekivano je da containeri rade kao `uid=1000(node)`.

## 5. Kubernetes deployment

Kubernetes manifesti nalaze se u:

```text
k8s/
```

Uključeni su:

- `Namespace`,
- `ConfigMap`,
- `Secret` primjer,
- `Deployment` za frontend, API, worker i Redis,
- `StatefulSet` za PostgreSQL,
- `Service` objekti,
- `Ingress`,
- readiness/liveness probeovi,
- resource requests/limits,
- `ServiceAccount`,
- RBAC,
- `NetworkPolicy`,
- Kustomize ulazna datoteka `k8s/kustomization.yaml`.

Primjer kreiranja Secreta prije deploya:

```bash
kubectl create namespace secure-ticketing
kubectl -n secure-ticketing create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='change-this-password'
```

Deploy:

```bash
kubectl apply -k k8s/
```

Provjera:

```bash
kubectl -n secure-ticketing get pods
kubectl -n secure-ticketing get svc
kubectl -n secure-ticketing get ingress
```

Port-forward validacija API-ja:

```bash
kubectl -n secure-ticketing port-forward svc/api 8080:8080
curl http://localhost:8080/readyz
```

Napomena: `Ingress` manifest je pripremljen za Kubernetes okruženje s instaliranim ingress controllerom. U lokalnoj validaciji funkcionalnost je moguće provjeriti i preko `port-forward` pristupa.

## 6. CI/CD i DevSecOps

CI/CD workflow nalazi se u:

```text
.github/workflows/ci-devsecops.yml
```

Workflow radi:

1. checkout koda,
2. instalaciju dependencyja,
3. sintaksnu validaciju Node.js servisa,
4. `npm audit` provjeru,
5. Docker build za aplikacijske servise,
6. Trivy config/IaC scan,
7. Trivy image scan,
8. blocking quality gate za `HIGH` i `CRITICAL` nalaze,
9. upload sigurnosnih artifacta,
10. opcionalni push imagea u GHCR za `main`/tagove.

Zadnja validacija:

- GitHub Actions run nakon završnih izmjena prošao je bez greške,
- artifacti su uploadani,
- Trivy config/IaC blocking gate je dodan i validiran lokalno.

Lokalna Trivy config provjera:

```bash
trivy config --severity HIGH,CRITICAL --exit-code 1 .
```

Ova naredba prolazi bez greške.

Detalji CI/CD procesa, politike tagiranja i mjerljivog napretka isporuke nalaze se u:

```text
docs/ci-cd.md
```

## 7. Rolling update i rollback

Postupak je dokumentiran u:

```text
docs/rolling-update-rollback.md
```

Osnovne naredbe:

```bash
kubectl -n secure-ticketing rollout status deployment/api
kubectl -n secure-ticketing rollout history deployment/api
kubectl -n secure-ticketing rollout undo deployment/api
```

Demonstrirano je:

- promjena image taga,
- uspješan rolling update,
- rollback na prethodnu verziju,
- validacija nakon rollbacka preko `/readyz` endpointa.

## 8. Troubleshooting / runbook

Runbook se nalazi u:

```text
docs/runbook.md
```

Pokriveni incidentni scenariji:

1. pad PostgreSQL baze,
2. loš image tag,
3. neispravan Kubernetes Secret ili environment varijabla.

Za svaki incident dokumentirani su:

- simptom,
- mogući uzrok,
- dijagnostičke naredbe,
- korektivna mjera,
- validacija nakon popravka,
- rollback/restore napomena gdje je primjenjivo.

## 9. Mapiranje na ishode I1-I6

| Ishod                                       | Implementacija                                                                         | Dokaz                                                                                              |
|---------------------------------------------|----------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| I1 — procjena upotrebe kontejnera i servisa | Servisna arhitektura, komunikacija i usporedba kontejnera s VM pristupom               | `docs/architecture.md`, `compose.yaml`, `k8s/`                                                     |
| I2 — sigurno upravljanje container imageima | Multi-stage build, non-root korisnik, minimalne runtime slike, Trivy scan              | `api/Dockerfile`, `frontend/Dockerfile`, `worker/Dockerfile`, `docs/security/image-scan-report.md` |
| I3 — ubrzana isporuka aplikacije            | GitHub Actions build/validacija/scan/publish workflow i dokumentirana metrika trajanja | `.github/workflows/ci-devsecops.yml`, `docs/ci-cd.md`                                              |
| I4 — DevSecOps metodologija                 | `npm audit`, Trivy image scan, Trivy config/IaC blocking gate, quality gate            | `.github/workflows/ci-devsecops.yml`, `docs/security image-scan-report.md`                         |
| I5 — rješavanje problema isporuke           | Incident runbook i rolling update/rollback dokumentacija                               | `docs/runbook.md`, `docs/rolling-update-rollback.md`                                               |
| I6 — orkestracija u složenijem scenariju    | Kubernetes manifesti, probes, resources, Ingress, Secret, RBAC, NetworkPolicy          | `k8s/`, `docs/deployment.md`                                                                       |
|---------------------------------------------|----------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|

## 10. Zaključak

Projekt pokriva tražene artefakte: source kod servisa, Dockerfileove, Compose lokalni stack, Kubernetes manifeste, sigurnosni scan imagea i konfiguracije, CI/CD pipeline, dokumentaciju za lokalno i produkcijsko pokretanje, runbook i finalnu checklistu.

Cilj projekta nije samo pokrenuti aplikaciju, nego demonstrirati cijeli DevSecOps tok isporuke: lokalni razvoj, build imagea, sigurnosne provjere, orkestraciju, rollout/rollback i troubleshooting.
