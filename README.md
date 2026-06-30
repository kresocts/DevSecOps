# Secure Event Ticketing Platform

Sample DevSecOps aplikacija za kontejnerizaciju, lokalni Compose stack, Kubernetes deployment manifeste i CI/CD sigurnosni workflow.

## Trenutni status repozitorija

Repozitorij nakon **Faze 7** sadrži aplikacijsku jezgru, Dockerfileove za aplikacijske servise, lokalni Docker/Podman Compose stack, Kubernetes manifeste, GitHub Actions DevSecOps workflow, dokumentiran rolling update/rollback postupak i incident runbook za:

- `frontend`
- `api`
- `worker`
- `postgres`
- `redis`
- CI/CD validaciju
- build container imagea
- Trivy image/config scan
- quality gate prije objave imagea
- rolling update i rollback postupak
- incident runbook za PostgreSQL pad, loš image tag i neispravan Secret/env varijablu
- finalnu arhitekturnu dokumentaciju i checklistu za PDF predaju

## Arhitektura aplikacije

Detaljna arhitektura i međuservisna komunikacija nalaze se u:

```text
docs/architecture.md
```


- `frontend` - web UI za pregled evenata i kupnju karata.
- `api` - REST API za evente, narudžbe i health/readiness provjere.
- `worker` - pozadinska obrada Redis queue poruka.
- `postgres` - trajna pohrana narudžbi, inicijalizacija kroz `infra/postgres/init.sql`.
- `redis` - queue/cache sloj za narudžbe.

Tok osnovnog workflowa:

```text
browser -> frontend -> /config -> browser -> api -> redis queue -> worker -> postgres
```

Frontend browseru vraća `API_BASE_URL`, browser direktno zove API, API stavlja narudžbu u Redis queue, a worker je obrađuje i sprema u PostgreSQL.

## Preduvjeti

Jedan od sljedećih alata:

- Docker Desktop s `docker compose`
- Docker Engine s Compose pluginom
- Podman s Compose kompatibilnim alatom

Za lokalni razvoj nije potrebno lokalno instalirati PostgreSQL ni Redis jer ih podiže Compose stack.

## Environment varijable

Kopiraj primjer konfiguracije:

```bash
cp .env.example .env
```

Važne varijable:

- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_HOST`
- `POSTGRES_PORT`
- `REDIS_HOST`
- `REDIS_PORT`
- `API_PORT`
- `FRONTEND_PORT`
- `API_BASE_URL`
- `QUEUE_NAME`
- `NODE_ENV`

Za lokalni demo `.env.example` koristi vrijednost `POSTGRES_PASSWORD=change_me_local`. Za produkciju se ta vrijednost ne smije koristiti; u kasnijoj Kubernetes/OpenShift fazi tajne idu kroz Secret.

## Lokalni startup

Pokretanje cijelog stacka jednom naredbom:

```bash
docker compose up --build
```

Pokretanje u pozadini:

```bash
docker compose up --build -d
```

Podman varijanta ovisi o instalaciji, ali najčešći oblik je:

```bash
podman compose up --build
```

## Lokalni shutdown

Zaustavljanje stacka bez brisanja PostgreSQL podataka:

```bash
docker compose down
```

Zaustavljanje i brisanje PostgreSQL volumea:

```bash
docker compose down -v
```

Volume koji čuva PostgreSQL podatke zove se:

```text
secure-event-ticketing_postgres_data
```

## Hot reload u razvoju

Compose koristi `dev` stage iz Dockerfileova za `api`, `frontend` i `worker`. Source folderi se mountaju u containere:

- `./api/src:/app/src:ro`
- `./frontend/src:/app/src:ro`
- `./worker/src:/app/src:ro`

Aplikacijski servisi se pokreću kroz `npm run dev`, odnosno `nodemon`, pa se promjene u `src` folderima automatski učitavaju gdje je to smisleno.

Ako mijenjaš `package.json` ili `package-lock.json`, ponovno izgradi image:

```bash
docker compose build --no-cache api frontend worker
```

## Health i readiness provjere

Kada je stack pokrenut:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:3000/healthz
curl http://localhost:3000/config
```

Očekivani API health odgovor:

```json
{"status":"ok","service":"api"}
```

Očekivani API readiness odgovor kada su PostgreSQL i Redis dostupni:

```json
{"status":"ready"}
```

Provjera statusa containera:

```bash
docker compose ps
```

Pregled logova:

```bash
docker compose logs -f api
docker compose logs -f worker
docker compose logs -f postgres
docker compose logs -f redis
```

## Osnovni workflow kupnje karte

1. Otvori frontend:

```text
http://localhost:3000
```

2. Odaberi event, upiši email i količinu, zatim klikni `Purchase`.

3. Ili napravi isti workflow preko API-ja:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
```

4. Provjeri da je worker obradio narudžbu i spremio je u PostgreSQL:

```bash
curl http://localhost:8080/tickets/orders
```

5. Ako narudžba ne dođe odmah, pogledaj worker logove:

```bash
docker compose logs -f worker
```

## Build produkcijskih imagea

Dockerfileovi i dalje imaju produkcijski `runtime` stage koji koristi `npm ci --omit=dev`, non-root korisnika i ne zapisuje tajne u image.

```bash
docker build -t ticketing-api:local ./api
docker build -t ticketing-frontend:local ./frontend
docker build -t ticketing-worker:local ./worker
```

Provjera da containeri ne rade kao root:

```bash
docker run --rm ticketing-api:local id
docker run --rm ticketing-frontend:local id
docker run --rm ticketing-worker:local id
```

Očekivano je `uid=1000(node)`.



## Incident runbook

Operativni runbook za incidente nalazi se u:

```text
docs/runbook.md
```

Pokriveni scenariji:

- pad PostgreSQL baze,
- loš image tag,
- neispravan Secret ili environment varijabla.

Svaki scenarij sadrži simptom, moguće uzroke, dijagnostičke naredbe, korektivnu mjeru, validaciju i rollback gdje je primjenjivo.

## CI/CD i DevSecOps

GitHub Actions workflow nalazi se u:

```text
.github/workflows/ci-devsecops.yml
```

Workflow radi:

- checkout repozitorija,
- instalaciju produkcijskih dependencyja za `api`, `frontend` i `worker`,
- minimalnu validaciju kroz `node -c`,
- `npm audit` gate za `HIGH` i veće nalaze,
- build container imagea,
- Trivy scan Kubernetes/Docker konfiguracije,
- Trivy scan container imagea,
- quality gate za `HIGH` i `CRITICAL` image ranjivosti,
- upload SARIF reporta kao GitHub Actions artifact,
- opcionalni push imagea u GHCR nakon prolaska gatea.

Detalji su u:

```text
docs/ci-cd.md
docs/security/image-scan-report.md
```

## Rolling update i rollback

Postupak promjene image taga, praćenja rollouta, pregleda rollout historyja i rollbacka dokumentiran je u:

```text
docs/rolling-update-rollback.md
```

Najvažnije naredbe:

```bash
kubectl -n secure-ticketing set image deployment/api api=<new-image-tag>
kubectl -n secure-ticketing rollout status deployment/api
kubectl -n secure-ticketing rollout history deployment/api
kubectl -n secure-ticketing rollout undo deployment/api
```

Image se objavljuje samo za `push` na `main`, `master` ili release tagove oblika `v*.*.*`. Pull request buildovi se buildaju i skeniraju, ali se ne objavljuju.

## Kubernetes deployment manifesti

Kubernetes manifesti nalaze se u folderu `k8s/`. Detaljne upute su u `docs/deployment.md`.

Minimalni redoslijed za deploy:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl -n secure-ticketing create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<strong-password>'
kubectl apply -k k8s/
```

Za lokalni cluster koji koristi lokalno buildane imagee prvo izgradi:

```bash
docker build -t ticketing-api:local ./api
docker build -t ticketing-frontend:local ./frontend
docker build -t ticketing-worker:local ./worker
```

Za produkcijski cluster zamijeni lokalne image tagove registry tagovima prije deploya. Secret se ne commita u repozitorij; `k8s/secret.example.yaml` je samo primjer.

## Testovi i validacija koda

Automatizirani testovi još ne postoje. Minimalna sintaktička validacija:

```bash
node -c api/src/server.js
node -c frontend/src/server.js
node -c worker/src/worker.js
```

Minimalna dependency validacija:

```bash
cd api && npm ci --omit=dev --ignore-scripts && cd ..
cd frontend && npm ci --omit=dev --ignore-scripts && cd ..
cd worker && npm ci --omit=dev --ignore-scripts && cd ..
```

## Finalna dokumentacija

Završni dokumentacijski artefakti nalaze se u:

```text
docs/architecture.md
docs/deployment.md
docs/ci-cd.md
docs/security/image-scan-report.md
docs/rolling-update-rollback.md
docs/runbook.md
docs/final-checklist.md
```

`docs/final-checklist.md` sadrži tablicu zahtjeva iz PDF-a, lokacije implementacije, status i napomene.


## Datoteke relevantne za trenutne faze

Faza 1 i 2:

- `compose.yaml`
- `.env.example`
- `api/Dockerfile`
- `frontend/Dockerfile`
- `worker/Dockerfile`

Faza 3:

- `k8s/`
- `docs/deployment.md`
- `docs/architecture.md`

Faza 4:

- `.github/workflows/ci-devsecops.yml`
- `docs/ci-cd.md`
- `docs/security/image-scan-report.md`

Faza 5:

- `docs/rolling-update-rollback.md`

Faza 6:

- `docs/runbook.md`

Faza 7:

- `docs/architecture.md`
- `docs/final-checklist.md`

## Poznati rizici nakon finalizacije

- Projekt nema pune unit/integration testove; CI zato koristi minimalni gate: `npm ci`, `node -c`, `npm audit` i Trivy scan.
- Compose koristi lokalnu demo lozinku iz `.env`; produkcijski deployment mora koristiti Kubernetes Secret.
- Ingress ovisi o dostupnom ingress controlleru u clusteru; lokalna validacija može se napraviti port-forwardom.
- K8s manifesti koriste lokalne `:local` image tagove za lab; za produkciju treba zamijeniti registry tagovima iz CI/CD pipelinea.
- Worker nema HTTP endpoint; u Kubernetes manifestima koristi se `exec` probe strategija.
