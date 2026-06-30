# Kubernetes deployment

Ovaj dokument pokriva **Fazu 3**: produkcijske Kubernetes manifeste za Secure Event Ticketing Platform.

Manifesti su u folderu `k8s/` i pokrivaju:

- `Namespace`
- `ConfigMap`
- primjer `Secret` objekta bez stvarne lozinke
- `Deployment` za `frontend`, `api`, `worker` i `redis`
- `StatefulSet` za `postgres`
- `Service` objekte za `frontend`, `api`, `postgres`, `redis`
- `Ingress` za vanjski pristup
- readiness/liveness probe
- resource requests/limits
- ServiceAccount i RBAC bez Kubernetes API ovlasti
- NetworkPolicy segmentaciju prometa

## Važna sigurnosna napomena

Ne zapisivati stvarnu PostgreSQL lozinku u Git niti u običan YAML manifest. Datoteka `k8s/secret.example.yaml` je samo primjer s placeholderom.

Preporučeni način kreiranja secreta:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl -n secure-ticketing create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<strong-password>'
```

Ako koristiš primjer:

```bash
cp k8s/secret.example.yaml k8s/secret.yaml
# zamijeni <replace-with-strong-password> stvarnom lokalnom/demo vrijednošću
kubectl apply -f k8s/secret.yaml
```

Datoteka `k8s/secret.yaml` je dodana u `.gitignore` i ne smije se commitati.

## Image tagovi

Manifesti koriste lokalne demo tagove:

```text
ticketing-api:local
ticketing-frontend:local
ticketing-worker:local
```

Za produkcijski cluster zamijeni ih tagovima iz registryja, npr. `ghcr.io/<org>/<image>:<tag>`.

Primjer izmjene prije deploya:

```bash
kubectl -n secure-ticketing set image deployment/api api=ghcr.io/<org>/ticketing-api:<tag>
kubectl -n secure-ticketing set image deployment/frontend frontend=ghcr.io/<org>/ticketing-frontend:<tag>
kubectl -n secure-ticketing set image deployment/worker worker=ghcr.io/<org>/ticketing-worker:<tag>
```

Ako koristiš lokalni Kubernetes koji vidi Docker Desktop image cache, prvo buildaj imagee iz root foldera:

```bash
docker build -t ticketing-api:local ./api
docker build -t ticketing-frontend:local ./frontend
docker build -t ticketing-worker:local ./worker
```

## Deploy

1. Kreiraj namespace:

```bash
kubectl apply -f k8s/namespace.yaml
```

2. Kreiraj Secret:

```bash
kubectl -n secure-ticketing create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD='<strong-password>'
```

3. Primijeni manifeste:

```bash
kubectl apply -k k8s/
```

4. Provjeri objekte:

```bash
kubectl -n secure-ticketing get pods
kubectl -n secure-ticketing get deploy,sts,svc,ingress
kubectl -n secure-ticketing get networkpolicy
kubectl -n secure-ticketing get sa,role,rolebinding
```

5. Čekaj rollout aplikacijskih deploymenta:

```bash
kubectl -n secure-ticketing rollout status deployment/api
kubectl -n secure-ticketing rollout status deployment/frontend
kubectl -n secure-ticketing rollout status deployment/worker
kubectl -n secure-ticketing rollout status deployment/redis
kubectl -n secure-ticketing rollout status statefulset/postgres
```

## Ingress

Ingress manifest koristi host:

```text
ticketing.example.com
```

Za lokalni test možeš dodati zapis u `/etc/hosts` prema IP adresi ingress controllera.

Primjer za lokalni `localhost` scenarij:

```text
127.0.0.1 ticketing.example.com
```

Ako tvoj cluster ne koristi `nginx` ingress class, promijeni `ingressClassName` u `k8s/ingress.yaml`.

## Validacija bez Ingressa

Ako Ingress još nije dostupan, koristi port-forward.

API:

```bash
kubectl -n secure-ticketing port-forward service/api 8080:8080
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

Frontend:

```bash
kubectl -n secure-ticketing port-forward service/frontend 3000:3000
curl http://localhost:3000/healthz
curl http://localhost:3000/config
```

Očekivano za frontend `/config` u Kubernetesu:

```json
{"apiBaseUrl":"/api"}
```

## Osnovni workflow kupnje karte

Kroz port-forward na API:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
```

Zatim provjeri obradu:

```bash
curl http://localhost:8080/tickets/orders
kubectl -n secure-ticketing logs deployment/worker
```

## Probes

- API readiness: `GET /readyz`, provjerava PostgreSQL i Redis.
- API liveness: `GET /healthz`.
- Frontend readiness/liveness: `GET /healthz`.
- Worker readiness: `exec` probe koja provjerava PostgreSQL i Redis konekciju.
- Worker liveness: `exec` provjera da Node worker proces postoji.
- PostgreSQL: `pg_isready`.
- Redis: `redis-cli ping`.

## Resources

Svi workloadi imaju `resources.requests` i `resources.limits`. Vrijednosti su male i prikladne za demo/lab okruženje. Za stvarnu produkciju potrebno ih je prilagoditi prema metrikama potrošnje.

## RBAC i ServiceAccount

Aplikacijski i data workloadi koriste ServiceAccount objekte s `automountServiceAccountToken: false`. Role ima prazna pravila (`rules: []`), što znači da workloadi nemaju Kubernetes API ovlasti.

## NetworkPolicy

NetworkPolicy manifesti rade sljedeće:

- default deny za ingress,
- default deny za egress,
- dopuštaju promet ingress controllera prema `frontend` i `api`,
- dopuštaju `api` i `worker` promet prema `postgres` i `redis`,
- dopuštaju DNS prema `kube-system` namespaceu.

Ako tvoj CNI plugin drugačije tretira egress prema Service IP adresama, možda će trebati prilagoditi egress pravila prema cluster CIDR-u ili CNI dokumentaciji.


## Rolling update i rollback

Detaljan postupak za promjenu image taga, praćenje rollout statusa, pregled rollout historyja, rollback i validaciju nalazi se u:

```text
docs/rolling-update-rollback.md
```

Kratki primjer za API:

```bash
kubectl -n secure-ticketing set image deployment/api api=<new-image-tag>
kubectl -n secure-ticketing rollout status deployment/api
kubectl -n secure-ticketing rollout history deployment/api
kubectl -n secure-ticketing rollout undo deployment/api
```

## Brisanje deploymenta

```bash
kubectl delete -k k8s/
kubectl -n secure-ticketing delete secret ticketing-secrets
kubectl delete namespace secure-ticketing
```

Ako želiš obrisati i PostgreSQL podatke, provjeri PVC i obriši ga svjesno:

```bash
kubectl -n secure-ticketing get pvc
kubectl -n secure-ticketing delete pvc -l app.kubernetes.io/component=postgres
```

### PostgreSQL CrashLoopBackOff kod prvog deploya

Ako `postgres-0` uđe u `CrashLoopBackOff`, prvo provjeri logove:

```bash
kubectl -n secure-ticketing logs postgres-0 --previous
kubectl -n secure-ticketing describe pod postgres-0
```

Ako log sadrži poruku poput:

```text
chmod: /var/lib/postgresql/data: Operation not permitted
initdb: error: could not change permissions of directory "/var/lib/postgresql/data": Operation not permitted
```

uzrok je dozvola na PVC-u. Manifest zato koristi:

- `PGDATA=/var/lib/postgresql/data/pgdata`, kako bi PostgreSQL podatke držao u poddirektoriju PVC-a,
- init container `postgres-volume-permissions`, koji priprema vlasništvo i dozvole volumea prije pokretanja glavnog PostgreSQL containera,
- glavni PostgreSQL container i dalje radi kao non-root korisnik `70`.

Primjena popravka na postojeći lokalni deployment:

```bash
kubectl apply -k k8s/
kubectl -n secure-ticketing delete pod postgres-0
kubectl -n secure-ticketing get pods -w
```

Ako je PVC ostao u lošem ili poluinicijaliziranom stanju, za lokalni reset demo baze možeš obrisati StatefulSet/PVC i ponovno primijeniti manifeste:

```bash
kubectl -n secure-ticketing delete statefulset postgres
kubectl -n secure-ticketing delete pvc postgres-data-postgres-0
kubectl apply -k k8s/
```

Ovo briše lokalne PostgreSQL podatke, pa se ne koristi za produkcijsku bazu bez backup/restore plana.

### Trivy KSV-0014 hardening za PostgreSQL i Redis

Ako `trivy config --severity HIGH,CRITICAL --exit-code 1 .` prijavi `KSV-0014` za `k8s/postgres.yaml` ili `k8s/redis.yaml`, manifesti trebaju imati `securityContext.readOnlyRootFilesystem: true` na containerima.

U ovim manifestima to je riješeno tako da:

- PostgreSQL glavni container koristi `readOnlyRootFilesystem: true`, ali i dalje ima writable PVC na `/var/lib/postgresql/data`, writable `emptyDir` na `/var/run/postgresql` i writable `emptyDir` na `/tmp`.
- PostgreSQL init container `postgres-volume-permissions` također koristi `readOnlyRootFilesystem: true`, ali može pisati u PostgreSQL PVC kako bi pripremio dozvole.
- Redis container koristi `readOnlyRootFilesystem: true`, a writable direktoriji su izdvojeni u `emptyDir` volume mountove na `/data` i `/tmp`.

Nakon ovog hardeninga ponovno validiraj:

```bash
trivy config --severity HIGH,CRITICAL --exit-code 1 .
kubectl apply -k k8s/
kubectl -n secure-ticketing rollout status statefulset/postgres --timeout=120s
kubectl -n secure-ticketing rollout status deployment/redis --timeout=120s
```
