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

## Trenutni poznati nalaz iz lokalne validacije

Tijekom lokalne validacije Faze 1 zabilježeno je da API dependency audit prijavljuje jedan `moderate` nalaz. Taj nalaz ne ruši trenutni CI gate jer je gate postavljen na `HIGH` i `CRITICAL`.

Prije finalne predaje treba ponovno pokrenuti CI i, ako nalaz i dalje postoji, evidentirati:

- paket koji uzrokuje nalaz,
- severity,
- postoji li siguran upgrade,
- odluku: popraviti odmah ili prihvatiti kao dokumentirani rizik za demo projekt.

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
docker build -t ticketing-api:local ./api
docker build -t ticketing-frontend:local ./frontend
docker build -t ticketing-worker:local ./worker
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
| Stvarni GitHub Actions run | čeka validaciju nakon pusha u GitHub |
| Finalni scan rezultati | dopuniti nakon prvog CI runa |
