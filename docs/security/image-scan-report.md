# Image scan report

Ovaj dokument opisuje kako se sigurnosni scan imagea provodi u CI/CD toku i gdje se čuvaju rezultati.

## Alat

Koristi se Trivy kroz GitHub Actions workflow:

```text
.github/workflows/ci-devsecops.yml
```

Skeniraju se:

- Kubernetes/Docker konfiguracija repozitorija,
- `api` container image,
- `frontend` container image,
- `worker` container image.

## Quality gate

Pipeline ruši build ako Trivy image scan pronađe ranjivost razine:

```text
HIGH
CRITICAL
```

Konfiguracija quality gatea u workflowu:

```yaml
severity: HIGH,CRITICAL
ignore-unfixed: true
exit-code: '1'
```

`MEDIUM` i `LOW` nalazi se ne koriste kao blokirajući gate u ovoj fazi, ali se trebaju pregledati i evidentirati kao tehnički dug.

## Nalazi iz validacije i korektivne mjere

Prva lokalna/GitHub Actions validacija Faze 4 pokazala je da image quality gate radi ispravno: build je zaustavljen kada je Trivy pronašao `HIGH` ranjivosti u runtime imageima.

Sažetak nalaza:

| Izvor nalaza | Severity | Korektivna mjera |
|---|---:|---|
| Alpine/OpenSSL paket u `node:20-alpine` runtime imageu | HIGH | dodan `apk upgrade --no-cache` u finalni runtime stage |
| Globalni `npm`/`yarn`/`corepack` alati iz Node base imagea | HIGH | uklonjeni iz finalnog runtime stagea jer nisu potrebni za pokretanje aplikacije |
| API `uuid` dependency `<11.1.1` | MEDIUM | nadograđen na `uuid@11.1.1` |

Odluka: ne uvoditi `.trivyignore` za ove nalaze jer postoje korektivne mjere. Quality gate ostaje postavljen na `HIGH`/`CRITICAL` i nastavlja blokirati objavu imagea dok nalaz nije saniran.

Finalna validacija je napravljena lokalno i kroz GitHub Actions. Lokalni Trivy image scanovi za `ticketing-api:local`, `ticketing-frontend:local` i `ticketing-worker:local` završili su s 0 `HIGH`/`CRITICAL` ranjivosti. GitHub Actions workflow run završio je sa statusom `Success` i generirao 4 artifacta.

## IaC/config scan hardening update

Lokalna validacija Faze 4 pronašla je `KSV-0014` `HIGH` misconfiguration nalaze u Kubernetes manifestima za PostgreSQL i Redis. Nalaz se odnosi na izostanak `securityContext.readOnlyRootFilesystem: true`.

Korektivna mjera je primijenjena u manifestima iz Faze 3:

| Datoteka | Nalaz | Korektivna mjera |
|---|---|---|
| `k8s/postgres.yaml` | PostgreSQL container bez read-only root filesystema | dodan `readOnlyRootFilesystem: true`; writable putanje izdvojene u PVC/`emptyDir` mountove |
| `k8s/postgres.yaml` | PostgreSQL init container bez read-only root filesystema | dodan `readOnlyRootFilesystem: true`; writable je samo PostgreSQL PVC i `/tmp` |
| `k8s/redis.yaml` | Redis container bez read-only root filesystema | dodan `readOnlyRootFilesystem: true`; writable putanje `/data` i `/tmp` izdvojene u `emptyDir` mountove |

Očekivana ponovna validacija:

```bash
trivy config --severity HIGH,CRITICAL --exit-code 1 .
```

Finalni rezultat: `trivy config --severity HIGH,CRITICAL --exit-code 1 .` završio je bez `HIGH`/`CRITICAL` misconfiguration nalaza; svi Dockerfile i Kubernetes targeti u summaryju imaju `0`.

## Lokacija izvještaja

GitHub Actions sprema SARIF izvještaje kao artifacts:

```text
trivy-config-sarif
trivy-api-image-sarif
trivy-frontend-image-sarif
trivy-worker-image-sarif
```

Ako je GitHub code scanning dostupan, workflow pokušava učitati iste SARIF nalaze u GitHub Security tab. Ako code scanning nije dostupan, artifacti su i dalje primarni izvor dokaza.

## Kako ručno pokrenuti scan

Nakon lokalnog builda:

```bash
docker build --no-cache -t ticketing-api:local ./api
docker build --no-cache -t ticketing-frontend:local ./frontend
docker build --no-cache -t ticketing-worker:local ./worker
```

Pokreni:

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-api:local
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-frontend:local
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-worker:local
```

Za spremanje rezultata u SARIF obliku:

```bash
mkdir -p reports
trivy image --format sarif --output reports/trivy-api-image.sarif ticketing-api:local
trivy image --format sarif --output reports/trivy-frontend-image.sarif ticketing-frontend:local
trivy image --format sarif --output reports/trivy-worker-image.sarif ticketing-worker:local
```

## Očekivana odluka kod nalaza

| Severity | Akcija |
|---|---|
| CRITICAL | blokirati objavu imagea, hitno popraviti ili dokumentirati iznimku |
| HIGH | blokirati objavu imagea, popraviti prije deploya |
| MEDIUM | analizirati i planirati popravak |
| LOW | evidentirati, ne blokira demo pipeline |
| UNKNOWN | ručno pregledati i odlučiti |

## Status za ovu fazu

| Stavka | Status |
|---|---|
| Trivy konfiguriran u CI/CD workflowu | gotovo |
| SARIF report artifacti konfigurirani | gotovo |
| HIGH/CRITICAL quality gate konfiguriran | gotovo |
| Automatski push imagea nakon prolaska gatea | gotovo, samo za `main`, `master` i tagove |
| Stvarni GitHub Actions run | gotovo, run `test #2` završio je sa `Success` |
| Finalni scan rezultati | gotovo, lokalni image scan i config scan bez `HIGH`/`CRITICAL` nalaza |
