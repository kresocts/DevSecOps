# CI/CD i DevSecOps workflow

Ova faza dodaje GitHub Actions workflow za validaciju, build container imagea, Trivy sigurnosno skeniranje i opcionalnu objavu imagea u GitHub Container Registry.

Workflow datoteka:

```text
.github/workflows/ci-devsecops.yml
```

## Kada se workflow pokreće

Workflow se pokreće na:

- `pull_request`, za validaciju promjena prije mergea,
- `push` na `main` ili `master`, za build, scan i opcionalnu objavu imagea,
- tagove u obliku `v*.*.*`, za release tagiranje imagea,
- ručno kroz `workflow_dispatch`.

## Poslovi u pipelineu

### 1. `validate`

Za svaki aplikacijski servis (`api`, `frontend`, `worker`) workflow radi:

```bash
npm ci --omit=dev --ignore-scripts
node -c <entrypoint>
npm audit --omit=dev --audit-level=high
```

Trenutni projekt nema automatizirane testove. Zato je quality gate u ovoj fazi minimalan: reproducibilan dependency install, sintaktička validacija i dependency audit za `HIGH` ili veće nalaze.

### 2. `trivy-repository-scan`

Trivy skenira Kubernetes/Docker konfiguraciju kroz `scan-type: config` i sprema SARIF izvještaj kao workflow artifact.

Izvještaj:

```text
trivy-config-sarif
```

Ako GitHub code scanning nije dostupan u repozitoriju, upload SARIF-a u Security tab može biti preskočen bez rušenja pipelinea. Artifact se svejedno sprema.

### 3. `build-scan-publish`

Za svaki aplikacijski servis workflow:

1. generira image tagove,
2. gradi image lokalno u GitHub runneru,
3. generira Trivy SARIF izvještaj,
4. sprema izvještaj kao artifact,
5. pokreće Trivy quality gate,
6. objavljuje image samo ako je event `push` na `main`, `master` ili release tag i ako quality gate prođe.

## Tagging politika

Image repository format:

```text
ghcr.io/<owner>/<repo>/<service>
```

Primjeri:

```text
ghcr.io/<owner>/<repo>/api:main-<short-sha>
ghcr.io/<owner>/<repo>/frontend:main-<short-sha>
ghcr.io/<owner>/<repo>/worker:main-<short-sha>
```

Svaki build dodatno dobiva immutable SHA tag:

```text
sha-<short-sha>
```

Za pull request se image builda i skenira, ali se ne objavljuje:

```text
pr-<number>-<short-sha>
```

Za Git tag `v1.0.0`, image dobiva tag:

```text
v1.0.0
```

## Politika objave imagea

Image se smije objaviti samo kada su ispunjeni svi uvjeti:

1. `validate` job je prošao,
2. image build je prošao,
3. Trivy image scan nije pronašao `HIGH` ili `CRITICAL` ranjivosti koje ruše gate,
4. workflow se izvršava na `push` prema `main`, `master` ili na release tag.

Pull request buildovi služe za provjeru i ne rade push u registry.

## Što se događa kod HIGH/CRITICAL nalaza

Ako Trivy pronađe `HIGH` ili `CRITICAL` nalaz u imageu:

- `Trivy image quality gate` step vraća exit code `1`,
- job pada,
- image se ne objavljuje,
- SARIF izvještaj ostaje spremljen kao artifact,
- ako je dostupno, nalaz se vidi i u GitHub Security / Code scanning sučelju.

## Gdje se čuvaju scan izvještaji

Workflow sprema SARIF izvještaje kao GitHub Actions artifacts:

```text
trivy-config-sarif
trivy-api-image-sarif
trivy-frontend-image-sarif
trivy-worker-image-sarif
```

Artifact retention je postavljen na 14 dana. Za završnu dokumentaciju ručno se može preuzeti relevantni SARIF ili summary i prenijeti nalaze u:

```text
docs/security/image-scan-report.md
```

## Registry autentikacija

Za GitHub Container Registry workflow koristi:

```text
secrets.GITHUB_TOKEN
```

Nije potrebno spremati dodatni registry password za osnovni GHCR scenarij u istom repozitoriju.

Ako se koristi Docker Hub, Quay ili drugi registry, treba dodati odgovarajuće GitHub Secrets, npr.:

```text
REGISTRY_USERNAME
REGISTRY_TOKEN
```

Tada treba prilagoditi `REGISTRY` i login step u workflowu.

## Lokalna validacija prije pusha

Minimalna lokalna validacija:

```bash
cd api && npm ci --omit=dev --ignore-scripts && npm audit --omit=dev --audit-level=high && node -c src/server.js && cd ..
cd frontend && npm ci --omit=dev --ignore-scripts && npm audit --omit=dev --audit-level=high && node -c src/server.js && cd ..
cd worker && npm ci --omit=dev --ignore-scripts && npm audit --omit=dev --audit-level=high && node -c src/worker.js && cd ..
```

Lokalni build imagea:

```bash
docker build -t ticketing-api:local ./api
docker build -t ticketing-frontend:local ./frontend
docker build -t ticketing-worker:local ./worker
```

Lokalni Trivy scan ako je Trivy instaliran:

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-api:local
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-frontend:local
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-worker:local
```
