# Incident runbook

Ovaj dokument pokriva **Fazu 6**: osnovni operativni runbook za incidente u Kubernetes/OpenShift deploymentu Secure Event Ticketing Platform aplikacije.

Runbook koristi namespace:

```bash
NS=secure-ticketing
```

Prije dijagnostike postavi varijablu:

```bash
NS=secure-ticketing
kubectl config current-context
kubectl -n $NS get pods
```

## Brzi pregled sustava

Koristi ove naredbe za početnu procjenu stanja:

```bash
kubectl -n $NS get deploy,sts,svc,pods
kubectl -n $NS get events --sort-by=.metadata.creationTimestamp | tail -50
kubectl -n $NS rollout status deployment/api --timeout=60s
kubectl -n $NS rollout status deployment/frontend --timeout=60s
kubectl -n $NS rollout status deployment/worker --timeout=60s
kubectl -n $NS rollout status statefulset/postgres --timeout=60s
```

Health provjera API-ja kroz port-forward:

```bash
kubectl -n $NS port-forward service/api 8080:8080
```

U drugom terminalu:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

`/healthz` potvrđuje da API proces radi. `/readyz` potvrđuje da API može pristupiti PostgreSQL-u i Redis-u.

---

## Scenarij 1: pad PostgreSQL baze

### Simptom

- `postgres-0` nije `1/1 Running`.
- API `/healthz` može vraćati OK, ali `/readyz` ne vraća `{"status":"ready"}`.
- API deployment može ostati bez dostupnih replika jer readiness probe ne prolazi.
- Worker može restartati ili ostati `0/1` jer ne može zapisivati narudžbe u bazu.

Tipičan prikaz:

```text
postgres-0   0/1   CrashLoopBackOff
api-*        0/1   Running
worker-*     0/1   Running
```

### Mogući uzroci

- Neispravan ili nedostajući `POSTGRES_PASSWORD` Secret.
- PVC problem ili neispravne dozvole na PostgreSQL volumeu.
- PostgreSQL init skripta nije uspješno izvršena.
- Nedovoljni resursi ili problem sa storage classom.
- Ručno obrisan Secret, ConfigMap ili PVC.

### Dijagnostičke naredbe

```bash
kubectl -n $NS get pod postgres-0
kubectl -n $NS describe pod postgres-0
kubectl -n $NS logs postgres-0 -c postgres
kubectl -n $NS logs postgres-0 -c postgres --previous
kubectl -n $NS get pvc
kubectl -n $NS describe pvc postgres-data-postgres-0
kubectl -n $NS get secret ticketing-secrets
kubectl -n $NS get configmap ticketing-config postgres-init
kubectl -n $NS get events --sort-by=.metadata.creationTimestamp | tail -50
```

Provjera API readinessa:

```bash
kubectl -n $NS port-forward service/api 8080:8080
curl http://localhost:8080/readyz
```

### Korektivna mjera

Ako Secret nedostaje, kreiraj ga ponovno:

```bash
kubectl -n $NS create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<strong-password>'
```

Ako je Secret postojao, ali je promijenjen, restartaj komponente koje ga koriste:

```bash
kubectl -n $NS rollout restart statefulset/postgres
kubectl -n $NS rollout restart deployment/api
kubectl -n $NS rollout restart deployment/worker
```

Ako je problem s dozvolama na lokalnom Docker Desktop PVC-u, provjeri da `k8s/postgres.yaml` sadrži `postgres-volume-permissions` init container i `PGDATA=/var/lib/postgresql/data/pgdata`. Zatim ponovno pokreni pod:

```bash
kubectl -n $NS delete pod postgres-0
kubectl -n $NS get pods -w
```

Ako je baza samo lokalni demo i podaci se smiju obrisati, napravi čisti reset baze:

```bash
kubectl -n $NS delete statefulset postgres
kubectl -n $NS delete pvc postgres-data-postgres-0
kubectl apply -k k8s/
kubectl -n $NS get pods -w
```

Ovo briše lokalne PostgreSQL podatke.

### Validacija da je problem riješen

```bash
kubectl -n $NS get pods
kubectl -n $NS rollout status statefulset/postgres --timeout=120s
kubectl -n $NS rollout status deployment/api --timeout=120s
kubectl -n $NS rollout status deployment/worker --timeout=120s
```

API readiness mora proći:

```bash
kubectl -n $NS port-forward service/api 8080:8080
curl http://localhost:8080/readyz
```

Očekivano:

```json
{"status":"ready"}
```

Provjeri osnovni workflow:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'

curl http://localhost:8080/tickets/orders
kubectl -n $NS logs deployment/worker --tail=50
```

### Rollback

Za PostgreSQL podatke nema jednostavan `kubectl rollout undo` ekvivalent koji vraća podatke. Ako je incident nastao zbog promjene manifesta, vrati prethodnu Git verziju manifesta i primijeni je:

```bash
git checkout <previous-commit> -- k8s/postgres.yaml k8s/postgres-init-configmap.yaml
kubectl apply -k k8s/
kubectl -n $NS rollout status statefulset/postgres --timeout=120s
```

Ako je incident posljedica lošeg Secret-a, rollback znači ponovno kreiranje Secret-a s prethodnom ispravnom vrijednošću i restart ovisnih workloadova.

---

## Scenarij 2: loš image tag

### Simptom

- Novi rollout zapne.
- Podovi su u statusu `ImagePullBackOff`, `ErrImagePull` ili `CrashLoopBackOff`.
- `kubectl rollout status deployment/<name>` ne završava uspješno.
- Dio starih replika može ostati aktivan, ali novi podovi ne postaju Ready.

Tipičan prikaz:

```text
api-xxxxx   0/1   ImagePullBackOff
```

### Mogući uzroci

- Image tag ne postoji u registryju.
- Image nije pushan u registry.
- Cluster nema pristup privatnom registryju ili nedostaje `imagePullSecret`.
- Image postoji, ali aplikacija u njemu ne starta.
- Image je za krivu arhitekturu ili krivi runtime.

### Dijagnostičke naredbe

```bash
kubectl -n $NS rollout status deployment/api --timeout=120s
kubectl -n $NS rollout history deployment/api
kubectl -n $NS get pods -l app.kubernetes.io/component=api
kubectl -n $NS describe deployment/api
kubectl -n $NS describe pod -l app.kubernetes.io/component=api
kubectl -n $NS logs deployment/api --tail=100
kubectl -n $NS get events --sort-by=.metadata.creationTimestamp | tail -50
```

Provjeri koji image deployment trenutno koristi:

```bash
kubectl -n $NS get deployment api \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="api")].image}{"\n"}'
```

### Korektivna mjera

Ako je tag pogrešan, vrati prethodnu verziju:

```bash
kubectl -n $NS rollout undo deployment/api
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Ako znaš točan ispravan tag, postavi ga eksplicitno:

```bash
kubectl -n $NS set image deployment/api api=<known-good-image-tag>
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Za frontend ili worker koristi isti obrazac:

```bash
kubectl -n $NS rollout undo deployment/frontend
kubectl -n $NS rollout undo deployment/worker
```

### Validacija da je problem riješen

```bash
kubectl -n $NS get pods -l app.kubernetes.io/component=api
kubectl -n $NS rollout history deployment/api
kubectl -n $NS port-forward service/api 8080:8080
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

Očekivano:

```json
{"status":"ok","service":"api"}
{"status":"ready"}
```

Nakon toga provjeri osnovni workflow kupnje karte:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'

curl http://localhost:8080/tickets/orders
```

### Rollback

Standardni rollback za deployment:

```bash
kubectl -n $NS rollout history deployment/api
kubectl -n $NS rollout undo deployment/api
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Rollback na konkretnu reviziju:

```bash
kubectl -n $NS rollout undo deployment/api --to-revision=<revision-number>
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Napomena: ako je deployment ranije primjenjivan s `kubectl apply`, Kubernetes može prikazati upozorenje da `last-applied-configuration` neće biti ažuriran. Za demo rollback to nije blokirajuće. Nakon rollbacka preporučljivo je uskladiti Git manifest s verzijom koja stvarno radi.

---

## Scenarij 3: neispravan Secret ili environment varijabla

### Simptom

- API `/healthz` vraća OK, ali `/readyz` ne vraća `ready`.
- API ili worker logovi pokazuju greške spajanja na PostgreSQL ili Redis.
- Podovi restartaju nakon promjene Secret-a ili ConfigMap-a.
- PostgreSQL radi, Redis radi, ali aplikacijski servisi ne mogu uspostaviti konekciju.

Mogući primjeri simptoma:

```text
password authentication failed
getaddrinfo ENOTFOUND postgres
ECONNREFUSED
Redis connection failed
```

### Mogući uzroci

- `POSTGRES_PASSWORD` u Secret-u ne odgovara stvarnoj lozinki baze.
- `POSTGRES_HOST`, `POSTGRES_PORT`, `REDIS_HOST` ili `REDIS_PORT` u ConfigMap-u su pogrešni.
- `QUEUE_NAME` nije usklađen između API-ja i workera.
- Secret je promijenjen, ali deploymenti nisu restartani.
- Secret nedostaje ili ima krivi key.

### Dijagnostičke naredbe

```bash
kubectl -n $NS get secret ticketing-secrets
kubectl -n $NS describe secret ticketing-secrets
kubectl -n $NS get configmap ticketing-config -o yaml
kubectl -n $NS describe deployment/api
kubectl -n $NS describe deployment/worker
kubectl -n $NS logs deployment/api --tail=100
kubectl -n $NS logs deployment/worker --tail=100
kubectl -n $NS get pods
```

Provjeri readiness:

```bash
kubectl -n $NS port-forward service/api 8080:8080
curl http://localhost:8080/readyz
```

Provjeri postoje li očekivani Secret keyevi bez ispisivanja vrijednosti:

```bash
kubectl -n $NS get secret ticketing-secrets -o jsonpath='{.data}'
```

Za lokalni demo možeš provjeriti samo da postoji key `POSTGRES_PASSWORD`. Ne ispisuj stvarne produkcijske tajne u terminal, screenshot ili dokumentaciju.

### Korektivna mjera

Ako Secret nedostaje:

```bash
kubectl -n $NS create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<strong-password>'
```

Ako Secret postoji, ali je kriv, zamijeni ga:

```bash
kubectl -n $NS delete secret ticketing-secrets
kubectl -n $NS create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<correct-password>'
```

Restartaj workloadove koji koriste Secret:

```bash
kubectl -n $NS rollout restart statefulset/postgres
kubectl -n $NS rollout restart deployment/api
kubectl -n $NS rollout restart deployment/worker
kubectl -n $NS rollout status statefulset/postgres --timeout=120s
kubectl -n $NS rollout status deployment/api --timeout=120s
kubectl -n $NS rollout status deployment/worker --timeout=120s
```

Ako je problem u ConfigMap-u, popravi `k8s/configmap.yaml`, primijeni promjenu i restartaj aplikacijske workloade:

```bash
kubectl apply -k k8s/
kubectl -n $NS rollout restart deployment/api
kubectl -n $NS rollout restart deployment/worker
kubectl -n $NS rollout status deployment/api --timeout=120s
kubectl -n $NS rollout status deployment/worker --timeout=120s
```

### Validacija da je problem riješen

```bash
kubectl -n $NS get pods
kubectl -n $NS port-forward service/api 8080:8080
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

Očekivano:

```json
{"status":"ready"}
```

Provjeri workflow:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'

curl http://localhost:8080/tickets/orders
kubectl -n $NS logs deployment/worker --tail=50
```

### Rollback

Ako je promjena bila u Git manifestu:

```bash
git checkout <previous-commit> -- k8s/configmap.yaml
kubectl apply -k k8s/
kubectl -n $NS rollout restart deployment/api deployment/worker
```

Ako je promjena bila u Secret-u, rollback znači ponovno kreiranje Secret-a s prethodnom ispravnom vrijednošću i restart workloadova koji ga koriste:

```bash
kubectl -n $NS delete secret ticketing-secrets
kubectl -n $NS create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<previous-known-good-password>'
kubectl -n $NS rollout restart statefulset/postgres deployment/api deployment/worker
```

---

## Završna validacija nakon incidenta

Nakon svakog incidenta napravi minimalnu završnu provjeru:

```bash
kubectl -n $NS get pods
kubectl -n $NS get deploy,sts,svc
kubectl -n $NS get events --sort-by=.metadata.creationTimestamp | tail -20
kubectl -n $NS port-forward service/api 8080:8080
```

U drugom terminalu:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
curl http://localhost:8080/tickets/orders
```

Incident se smatra riješenim kada:

- svi ključni podovi imaju status `Running`,
- API `/readyz` vraća `ready`,
- worker obrađuje narudžbu,
- narudžba se vidi kroz `/tickets/orders`,
- nema novih relevantnih error eventova.
