# Architecture

Secure Event Ticketing Platform je višeslojna demo aplikacija za DevOps/DevSecOps laboratorij. Cilj arhitekture je jasno razdvojiti web sloj, API sloj, asinkronu obradu, trajnu pohranu i queue/cache sloj.


## Zašto kontejneri umjesto virtualnih mašina

| Kriterij | Kontejneri | Virtualne mašine |
|---|---|---|
| Pokretanje | Brže pokretanje jer dijele kernel hosta | Sporije pokretanje jer svaki VM ima vlastiti OS |
| Potrošnja resursa | Manja potrošnja memorije i diska | Veća potrošnja zbog cijelog guest OS-a |
| CI/CD | Pogodni za automatizirani build, scan, tagiranje i deploy | Teže ih je brzo graditi, skenirati i distribuirati |
| Izolacija | Dovoljna za aplikacijske servise uz non-root, minimalne slike i NetworkPolicy | Jača izolacija, ali uz veći operativni trošak |
| Primjena u ovom projektu | Prikladni za frontend, API, worker, Redis i PostgreSQL u DevSecOps pipelineu | Pretjerani za ovu višeslojnu aplikaciju i sporiji za lokalni razvoj |

Za Secure Event Ticketing Platform kontejneri su bolji izbor jer omogućuju standardizirano lokalno pokretanje kroz Compose, ponovljiv build i skeniranje imagea kroz CI/CD te jednostavan prijenos iste aplikacije na Kubernetes/OpenShift. Virtualne mašine bi dale jaču izolaciju, ali bi povećale vrijeme isporuke, potrošnju resursa i kompleksnost održavanja za ovaj projektni scenarij.

## Servisi

| Servis | Uloga | Runtime / image | Port | State |
|---|---|---|---:|---|
| `frontend` | Web UI i `/config` endpoint za browser konfiguraciju | Node.js + Express | `3000` | stateless |
| `api` | REST API za evente, narudžbe, health i readiness | Node.js + Express | `8080` | stateless |
| `worker` | Pozadinska obrada narudžbi iz Redis queuea | Node.js | nema HTTP port | stateless proces |
| `postgres` | Trajna pohrana obrađenih narudžbi | PostgreSQL 16 Alpine | `5432` | stateful |
| `redis` | Queue/cache sloj za narudžbe | Redis 7 Alpine | `6379` | ephemeral queue/cache |

## Struktura repozitorija

```text
.
├── api/                    # REST API service
├── frontend/               # Web frontend service
├── worker/                 # Background worker service
├── infra/postgres/         # PostgreSQL init SQL
├── k8s/                    # Kubernetes manifests
├── docs/                   # Deployment, CI/CD, runbook and architecture docs
├── compose.yaml            # Local developer stack
├── .env.example            # Local environment template
└── .github/workflows/      # CI/CD DevSecOps workflow
```

## Logički tok zahtjeva

```text
Browser
  |
  | GET /
  v
frontend :3000
  |
  | GET /config -> returns API_BASE_URL
  v
Browser
  |
  | POST /tickets/purchase
  v
api :8080
  |
  | LPUSH ticket_orders
  v
redis :6379
  |
  | BRPOP ticket_orders
  v
worker
  |
  | INSERT ticket_orders
  v
postgres :5432
```

Nakon obrade, narudžbe se čitaju kroz:

```text
GET /tickets/orders
```

API vraća zadnjih 50 narudžbi spremljenih u PostgreSQL.

## Frontend komunikacija s API-jem

Frontend ne hardkodira API adresu u HTML-u. Server izlaže:

```text
GET /config
```

koji vraća:

```json
{"apiBaseUrl":"http://localhost:8080"}
```

U lokalnom Compose okruženju browser direktno zove `http://localhost:8080`. U Kubernetes okruženju `API_BASE_URL` je postavljen na `/api`, što odgovara Ingress pravilu.

## API endpointi

| Endpoint | Metoda | Namjena |
|---|---|---|
| `/healthz` | `GET` | osnovni liveness health API servisa |
| `/readyz` | `GET` | readiness provjera PostgreSQL i Redis konekcije |
| `/events` | `GET` | lista demo evenata |
| `/tickets/purchase` | `POST` | validira narudžbu i stavlja je u Redis queue |
| `/tickets/orders` | `GET` | čita obrađene narudžbe iz PostgreSQL-a |

API podržava i `/api/...` prefiks za Ingress scenarij; zahtjev `/api/events` interno se mapira na `/events`.

## Redis queue

Queue ime dolazi iz environment varijable:

```text
QUEUE_NAME=ticket_orders
```

API koristi `LPUSH`, a worker koristi blokirajući `BRPOP`. Time se demonstrira asinkrona obrada bez potrebe za kompleksnim message brokerom.

## PostgreSQL pohrana

Inicijalna shema je u:

```text
infra/postgres/init.sql
```

Tablica:

```text
ticket_orders
```

sadrži `order_id`, `event_id`, `customer_email`, `quantity`, `status` i `created_at`.

U Compose okruženju podaci se čuvaju u named volumeu:

```text
secure-event-ticketing_postgres_data
```

U Kubernetes okruženju PostgreSQL radi kao `StatefulSet` s PVC-om.

## Environment varijable

| Varijabla | Koristi | Opis |
|---|---|---|
| `POSTGRES_DB` | API, worker, PostgreSQL | naziv baze |
| `POSTGRES_USER` | API, worker, PostgreSQL | korisnik baze |
| `POSTGRES_PASSWORD` | API, worker, PostgreSQL | lozinka, u produkciji Secret |
| `POSTGRES_HOST` | API, worker | host PostgreSQL servisa |
| `POSTGRES_PORT` | API, worker | port PostgreSQL servisa |
| `REDIS_HOST` | API, worker | host Redis servisa |
| `REDIS_PORT` | API, worker | port Redis servisa |
| `API_PORT` | API | HTTP port API-ja |
| `FRONTEND_PORT` | frontend | HTTP port frontenda |
| `API_BASE_URL` | frontend | API URL koji frontend vraća browseru |
| `QUEUE_NAME` | API, worker | Redis queue/list name |
| `NODE_ENV` | frontend, API, worker | runtime mode |


### Napomena o lokalnim fallback vrijednostima

Lokalni fallbackovi poput `change_me_local` koriste se samo za developer environment. U Kubernetes deploymentu vrijednosti dolaze iz Secreta i ne smiju se hardkodirati u produkcijskom okruženju.

## Lokalno okruženje

`compose.yaml` pokreće svih pet servisa:

- `frontend`,
- `api`,
- `worker`,
- `postgres`,
- `redis`.

Compose koristi `dev` stage iz Dockerfileova za aplikacijske servise, mounta `src` direktorije kao read-only volume i pokreće `npm run dev` preko `nodemon` gdje je smisleno.

## Kubernetes / produkcijsko okruženje

Kubernetes manifesti u `k8s/` uključuju:

- `Namespace`,
- `ConfigMap`,
- `Secret` primjer i dokumentirani način kreiranja stvarnog Secreta,
- `Deployment` za `frontend`, `api`, `worker`, `redis`,
- `StatefulSet` za `postgres`,
- `Service` objekte,
- `Ingress`,
- readiness i liveness probe,
- resource requests/limits,
- ServiceAccount/RBAC,
- NetworkPolicy.

Detalji deploya su u:

```text
docs/deployment.md
```

## Sigurnosne kontrole

| Kontrola | Implementacija |
|---|---|
| Non-root runtime | Dockerfileovi koriste `USER node`; K8s securityContext koristi `runAsNonRoot` |
| Minimalniji runtime image | multi-stage Dockerfileovi s runtime stageom i produkcijskim dependencyjima |
| Bez hardkodiranih produkcijskih tajni | `.env.example` je samo lokalni primjer; K8s koristi Secret |
| Read-only root filesystem | K8s workloadi koriste `readOnlyRootFilesystem: true` gdje je primjenjivo |
| Capabilities drop | K8s workloadi dropaju Linux capabilities |
| RBAC least privilege | ServiceAccount token nije automatski mountan; Role nema API ovlasti |
| Network segmentation | default deny NetworkPolicy + eksplicitno dopušteni tokovi |
| Image scanning | Trivy image scan u GitHub Actions workflowu |
| IaC scanning | Trivy config scan za Docker/K8s konfiguraciju |
| Quality gate | HIGH/CRITICAL nalazi blokiraju image publish |

## Observability i troubleshooting

Aplikacija koristi standardni stdout/stderr logging i health/readiness endpoint/probe pristup.

Korisne naredbe:

```bash
# Compose
docker compose ps
docker compose logs -f api
docker compose logs -f worker

# Kubernetes
kubectl -n secure-ticketing get pods
kubectl -n secure-ticketing describe pod <pod>
kubectl -n secure-ticketing logs deployment/api
kubectl -n secure-ticketing logs deployment/worker
kubectl -n secure-ticketing rollout status deployment/api
kubectl -n secure-ticketing events --sort-by=.lastTimestamp
```

Incidentni postupci su u:

```text
docs/runbook.md
```

Rolling update i rollback postupci su u:

```text
docs/rolling-update-rollback.md
```

## Glavne pretpostavke

- PostgreSQL i Redis koriste službene upstream imagee, a aplikacijski servisi imaju vlastite Dockerfileove.
- Lokalni `.env.example` sadrži demo vrijednosti i ne predstavlja produkcijski secret management.
- U produkciji se lokalni `:local` image tagovi zamjenjuju registry tagovima iz CI/CD pipelinea.
- Ingress manifest pretpostavlja `nginx` ingress class; za OpenShift bi se mogao koristiti Route umjesto Ingressa.
