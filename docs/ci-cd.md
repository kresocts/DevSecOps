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

Trivy skenira Kubernetes/Docker konfiguraciju kroz `scan-type: config`, sprema SARIF izvještaj kao workflow artifact i zatim pokreće zaseban blocking quality gate za `HIGH`/`CRITICAL` IaC/config nalaze.

Izvještaj:

```text
trivy-config-sarif
```

Ako GitHub code scanning nije dostupan u repozitoriju, upload SARIF-a u Security tab može biti preskočen bez rušenja pipelinea. Artifact se svejedno sprema. Blocking gate ostaje aktivan i ruši job ako Trivy pronađe `HIGH` ili `CRITICAL` misconfiguration.

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


## Validacija GitHub Actions workflowa

Zadnji validirani GitHub Actions run završen je uspješno. Workflow je validirao npm instalaciju, syntax check, npm audit, Docker build, Trivy skeniranje i upload sigurnosnih artifacta.

`actions/upload-artifact@v7` ostaje u workflowu jer je upload artifacta uspješno izvršen i u runu su bila dostupna 4 artifacta. Verzija nije mijenjana jer nema dokaza da taj korak ruši workflow.

`docker/login-action` je ažuriran s `v3` na `v4` zato što je prethodni run prikazao Node.js 20 deprecation warning za `docker/login-action@v3`, a `v4` koristi Node.js 24 runtime. To uklanja poznati warning bez promjene funkcionalnosti login koraka.

## Mjerljiv napredak brzine isporuke

Prije automatizacije, validacija projekta zahtijevala je ručno pokretanje više odvojenih koraka: instalaciju dependencyja za svaki servis, syntax check, npm audit, Docker build za API/frontend/worker, Trivy image scan, Trivy config scan i provjeru Kubernetes manifestâ.

CI/CD workflow te korake izvodi standardizirano u jednom pipelineu. Zadnji uspješni GitHub Actions run trajao je približno 1 minutu i 20 sekundi i generirao sigurnosne artifacte, čime se ručni niz provjera zamjenjuje ponovljivim i bržim procesom.

Time je isporuka ubrzana jer se svaki push automatski validira kroz isti build, security scan i quality gate, umjesto da se provjere izvode ručno i nekonzistentno.

## Lokalna validacija prije pusha

Minimalna lokalna validacija:

```bash
cd api && npm ci --omit=dev --ignore-scripts && npm audit --omit=dev --audit-level=high && node -c src/server.js && cd ..
cd frontend && npm ci --omit=dev --ignore-scripts && npm audit --omit=dev --audit-level=high && node -c src/server.js && cd ..
cd worker && npm ci --omit=dev --ignore-scripts && npm audit --omit=dev --audit-level=high && node -c src/worker.js && cd ..
```

Lokalni build imagea:

```bash
docker build --no-cache -t ticketing-api:local ./api
docker build --no-cache -t ticketing-frontend:local ./frontend
docker build --no-cache -t ticketing-worker:local ./worker
```

Lokalni Trivy scan ako je Trivy instaliran. Nakon promjene Dockerfilea preporučeno je koristiti `--no-cache` kod builda kako se ne bi skenirao stari cacheirani sloj:

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-api:local
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-frontend:local
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-worker:local
```

Lokalni IaC/config quality gate:

```bash
trivy config --severity HIGH,CRITICAL --exit-code 1 .
```


## Korektivne mjere nakon prve CI validacije

Prva validacija Faze 4 pokazala je da GitHub Actions `validate` poslovi prolaze, ali image quality gate ruši build jer Trivy pronalazi `HIGH` ranjivosti u finalnom runtime imageu. Nalozi su bili iz dva izvora:

- Alpine/OpenSSL paket u base imageu,
- globalni `npm`/`yarn`/`corepack` alati iz službenog Node imagea, koji nisu potrebni za runtime izvršavanje aplikacije.

Korekcija je napravljena u sva tri Dockerfilea:

- finalni runtime stage pokreće `apk upgrade --no-cache`,
- nakon instalacije sistemskih paketa uklanjaju se `npm`, `npx`, `corepack` i `yarn` iz runtime stagea,
- dependency install i dalje ostaje u `deps`/`dev` stageovima, pa build i hot reload nisu uklonjeni.

Dodatno je API `uuid` dependency nadograđen sa `10.x` na `11.1.1` kako bi se uklonio raniji `moderate` audit nalaz bez promjene aplikacijske funkcionalnosti.

GitHub Actions deprecation warning za Node 20 action runtime smanjen je nadogradnjom službenih GitHub Actions verzija na generaciju koja koristi Node 24 runtime:

- `actions/checkout@v7`,
- `actions/setup-node@v6`,
- `actions/upload-artifact@v7`.

## Finalna validacija pipelinea

GitHub Actions workflow je validiran stvarnim `push` runom na `main` grani. Run `test #2` završio je sa statusom `Success`, svi matrix validate jobovi su prošli, Trivy repository/IaC scan je prošao, sva tri build-scan-publish joba su prošla i kreirana su 4 artifacta.

Preostali warning za `docker/login-action@v3` odnosi se na GitHub Actions Node runtime deprecation poruku. Warning nije blokirao pipeline niti mijenja DevSecOps quality gate rezultat.
