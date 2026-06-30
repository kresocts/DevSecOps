# Rolling update i rollback

Ovaj dokument pokriva **Fazu 5**: demonstraciju promjene image taga, praćenje rollouta, pregled rollout historyja, rollback na prethodnu verziju i validaciju nakon promjene.

Primjeri koriste Kubernetes namespace:

```bash
NS=secure-ticketing
```

Za lokalni Docker Desktop cluster manifesti koriste lokalne image tagove:

```text
ticketing-api:local
ticketing-frontend:local
ticketing-worker:local
```

U produkciji se očekuju registry tagovi, npr.:

```text
ghcr.io/<owner>/<repo>/api:main-<short-sha>
ghcr.io/<owner>/<repo>/frontend:main-<short-sha>
ghcr.io/<owner>/<repo>/worker:main-<short-sha>
```

## Preduvjeti

Deployment mora već biti primijenjen i spreman:

```bash
kubectl -n $NS get pods
kubectl -n $NS rollout status deployment/api
kubectl -n $NS rollout status deployment/frontend
kubectl -n $NS rollout status deployment/worker
```

Očekivano je da su `api`, `frontend`, `worker`, `postgres` i `redis` podovi u stanju `Running`, a aplikacijski workloadi `Ready`.

## 1. Pregled trenutnih image tagova

```bash
kubectl -n $NS get deployment api -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n $NS get deployment frontend -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n $NS get deployment worker -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Spremi trenutne vrijednosti ako želiš ručno vratiti image:

```bash
API_OLD_IMAGE=$(kubectl -n $NS get deployment api -o jsonpath='{.spec.template.spec.containers[0].image}')
FRONTEND_OLD_IMAGE=$(kubectl -n $NS get deployment frontend -o jsonpath='{.spec.template.spec.containers[0].image}')
WORKER_OLD_IMAGE=$(kubectl -n $NS get deployment worker -o jsonpath='{.spec.template.spec.containers[0].image}')
```

## 2. Rolling update API servisa

Primjer s novim API image tagom:

```bash
API_NEW_IMAGE=ticketing-api:v2
```

Za lokalni Docker Desktop demo prvo buildaj image koji Kubernetes može dohvatiti iz lokalnog image cachea:

```bash
docker build -t $API_NEW_IMAGE ./api
```

Označi razlog promjene i promijeni image:

```bash
kubectl -n $NS annotate deployment/api kubernetes.io/change-cause="api image update to $API_NEW_IMAGE" --overwrite
kubectl -n $NS set image deployment/api api=$API_NEW_IMAGE
```

Prati rollout:

```bash
kubectl -n $NS rollout status deployment/api --timeout=120s
kubectl -n $NS get pods -l app.kubernetes.io/component=api -w
```

Deployment koristi rolling update strategiju s `maxUnavailable: 0` i `maxSurge: 1`, pa tijekom uredne promjene ne bi trebao ostati bez dostupnog API poda.

## 3. Rollout history

Pregled povijesti:

```bash
kubectl -n $NS rollout history deployment/api
```

Detalji pojedine revizije:

```bash
kubectl -n $NS rollout history deployment/api --revision=1
kubectl -n $NS rollout history deployment/api --revision=2
```

Provjera aktivnog imagea:

```bash
kubectl -n $NS get deployment api -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## 4. Validacija nakon rolling updatea

Port-forward API servisa:

```bash
kubectl -n $NS port-forward service/api 8080:8080
```

U drugom terminalu:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

Očekivano:

```json
{"status":"ok","service":"api"}
{"status":"ready"}
```

End-to-end workflow:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'

curl http://localhost:8080/tickets/orders
kubectl -n $NS logs deployment/worker --tail=50
```

Očekivano je da narudžba dobije status `processed`, a worker log sadrži `Order processed`.

## 5. Rollback API servisa

Rollback na prethodnu reviziju:

```bash
kubectl -n $NS rollout undo deployment/api
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Rollback na konkretnu reviziju:

```bash
kubectl -n $NS rollout history deployment/api
kubectl -n $NS rollout undo deployment/api --to-revision=<revision-number>
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Validacija nakon rollbacka:

```bash
kubectl -n $NS get deployment api -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

Ako koristiš port-forward, ostavi ga aktivnim u posebnom terminalu ili ga ponovno pokreni.

## 6. Rolling update frontend servisa

Primjer:

```bash
FRONTEND_NEW_IMAGE=ticketing-frontend:v2

docker build -t $FRONTEND_NEW_IMAGE ./frontend
kubectl -n $NS annotate deployment/frontend kubernetes.io/change-cause="frontend image update to $FRONTEND_NEW_IMAGE" --overwrite
kubectl -n $NS set image deployment/frontend frontend=$FRONTEND_NEW_IMAGE
kubectl -n $NS rollout status deployment/frontend --timeout=120s
kubectl -n $NS rollout history deployment/frontend
```

Validacija:

```bash
kubectl -n $NS port-forward service/frontend 3000:3000
curl http://localhost:3000/healthz
curl http://localhost:3000/config
```

Rollback:

```bash
kubectl -n $NS rollout undo deployment/frontend
kubectl -n $NS rollout status deployment/frontend --timeout=120s
```

## 7. Rolling update worker servisa

Primjer:

```bash
WORKER_NEW_IMAGE=ticketing-worker:v2

docker build -t $WORKER_NEW_IMAGE ./worker
kubectl -n $NS annotate deployment/worker kubernetes.io/change-cause="worker image update to $WORKER_NEW_IMAGE" --overwrite
kubectl -n $NS set image deployment/worker worker=$WORKER_NEW_IMAGE
kubectl -n $NS rollout status deployment/worker --timeout=120s
kubectl -n $NS rollout history deployment/worker
```

Worker nema HTTP port. Njegova readiness probe koristi `exec` provjeru PostgreSQL i Redis konekcije, a liveness probe provjerava da Node worker proces postoji.

Validacija workera radi se kroz queue workflow:

```bash
kubectl -n $NS port-forward service/api 8080:8080
```

U drugom terminalu:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'

curl http://localhost:8080/tickets/orders
kubectl -n $NS logs deployment/worker --tail=50
```

Rollback:

```bash
kubectl -n $NS rollout undo deployment/worker
kubectl -n $NS rollout status deployment/worker --timeout=120s
```

## 8. Namjerno loš image tag i oporavak

Za demonstraciju incidenta može se postaviti nepostojeći image tag:

```bash
kubectl -n $NS annotate deployment/api kubernetes.io/change-cause="demo bad api image tag" --overwrite
kubectl -n $NS set image deployment/api api=ticketing-api:does-not-exist
kubectl -n $NS rollout status deployment/api --timeout=60s
```

Očekivano je da rollout ne uspije. Dijagnostika:

```bash
kubectl -n $NS get pods -l app.kubernetes.io/component=api
kubectl -n $NS describe deployment/api
kubectl -n $NS get events --sort-by=.lastTimestamp | tail -n 30
```

Rollback:

```bash
kubectl -n $NS rollout undo deployment/api
kubectl -n $NS rollout status deployment/api --timeout=120s
```

Validacija:

```bash
kubectl -n $NS get pods -l app.kubernetes.io/component=api
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

## 9. Što priložiti kao dokaz za PDF

Za dokaz Faze 5 možeš spremiti terminal output ili screenshotove za:

```bash
kubectl -n secure-ticketing rollout status deployment/api
kubectl -n secure-ticketing rollout history deployment/api
kubectl -n secure-ticketing rollout undo deployment/api
curl http://localhost:8080/readyz
curl http://localhost:8080/tickets/orders
```

Dovoljno je demonstrirati jedan servis, obično `api`, jer on ima najjasniju readiness provjeru i end-to-end validaciju. Za potpuniji dokaz može se ponoviti kraća verzija za `frontend` i `worker`.
