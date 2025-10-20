# test-initdb

Minimal CockroachDB deployment on Minikube using a SQL "initdb" style bootstrap to create a `testdb` database.

---

## Architecture Overview

This repository provides two layers:

1. CockroachDB Infrastructure (single-node, insecure, dev only)
	* Namespace: `crdb`
	* StatefulSet: `cockroachdb` (runs `start-single-node` so it self-initializes)
	* Services:
	  - `cockroachdb` (headless for stable pod DNS)
	  - `cockroachdb-public` (client & UI access)
	* Bootstrap SQL Job: `bootstrap-sql` applies `init.sql` (idempotent) creating logical database `testdb`.
	* Kustomize generates a ConfigMap from `init.sql` (`bootstrap-sql`).

2. PIM Backend Application (Express + Prisma)
	* Namespace: `pim`
	* Deployment: `pim-backend-api` consuming `DATABASE_URL` that points at `testdb` via `cockroachdb-public` service.
	* On startup the container runs Prisma migrations / schema push inside `testdb` (tables & indexes).

Separation of concerns:
* Infrastructure layer guarantees the CockroachDB node and the logical database exist.
* Application layer owns schema objects (tables) within that logical database.

Flow diagram (conceptual):
```
init.sql -> ConfigMap -> bootstrap-sql Job --> creates database testdb
															 |
															 v
										pim-backend-api Deployment (Prisma) -> creates/updates tables in testdb
```

## Contents

`crdb-minikube/` holds a tiny, ordered set of Kubernetes manifests plus an `init.sql` file:

```
crdb-minikube/
	namespace.yaml                   # Creates dedicated namespace `crdb`
	services.yaml                    # Headless service for pod DNS + public service for client/UI access
	cockroachdb-statefulset.yaml     # Single-node CockroachDB in insecure mode (auto-initializes)
	bootstrap-sql-job.yaml           # Executes the SQL against the running node
	init.sql                         # Source SQL file (ConfigMap generated via Kustomize)
	kustomization.yaml               # Generates ConfigMap from init.sql + applies manifests
```

## Quick start

```bash
minikube start --cpus=4 --memory=6144

# Apply base objects
kubectl apply -f crdb-minikube/namespace.yaml
kubectl apply -f crdb-minikube/services.yaml
kubectl apply -f crdb-minikube/cockroachdb-statefulset.yaml

# Wait for CockroachDB pod readiness
kubectl -n crdb rollout status statefulset/cockroachdb

# Initialize the cluster (system ranges)
# (Single-node mode auto-initializes; cluster init job removed)

# Create the testdb database
kubectl apply -k crdb-minikube

# Verify
kubectl -n crdb exec -it statefulset/cockroachdb -- \
	/cockroach/cockroach sql --insecure --host=cockroachdb-public.crdb.svc.cluster.local \
	-e "SHOW DATABASES;"

# (Optional) CockroachDB Web UI
kubectl -n crdb port-forward svc/cockroachdb-public 8080:8080
# Visit http://localhost:8080
```

Expected databases: `system`, `defaultdb`, `postgres`, and `testdb`.

If the bootstrap Job runs after the DB pod is ready, `testdb` will appear. The Job is idempotent; re-running kustomize apply is safe.

## Notes

* Single-node uses `start-single-node` which performs cluster init automatically; thus no separate `cockroach init` Job.
* This is intentionally single-node and `--insecure` for rapid local testing; do **not** use insecure mode in shared or production clusters.
* Scale to 3 nodes: switch command back to `start` (not `start-single-node`), set `replicas: 3`, add explicit `--listen-addr`, `--sql-addr`, `--advertise-addr` and keep a `--join` list; then reintroduce an init Job to run once.
* Secure mode: create certificates (node + client) as Kubernetes Secrets and replace `--insecure` with `--certs-dir=/cockroach/certs`.
* Editing `init.sql` and re-applying (`kubectl apply -k crdb-minikube`) regenerates the ConfigMap (name stable via `disableNameSuffixHash`).
* Add more SQL (users, grants) in `init.sql`; keep it idempotent (`IF NOT EXISTS`) for safe re-sync.

## PIM Backend (Express + Prisma) Deployment

Kustomize manifests live under `crdb-minikube/pim-backend/`:

```
crdb-minikube/pim-backend/
	namespace.yaml          # pim namespace
	configmap.yaml          # Non-sensitive env (DATABASE_URL, PORT, seed toggle)
	deployment.yaml         # Single replica backend, probes, resource requests
	service.yaml            # ClusterIP service on port 3000
	kustomization.yaml      # Aggregates the above
```

### Build the image locally (Minikube Docker)
```bash
eval "$(minikube docker-env)"   # or: minikube image build -t pim-backend-api:dev -f Dockerfile /path/to/pim-backend-api
minikube image build -t pim-backend-api:dev -f /home/jkicklighter@internal.rotarycorp.com/Documents/Maxpower/backend/pim-backend-api/Dockerfile /home/jkicklighter@internal.rotarycorp.com/Documents/Maxpower/backend/pim-backend-api
```

### Apply backend manifests manually
```bash
kubectl apply -k crdb-minikube/pim-backend
kubectl -n pim get pods
kubectl -n pim logs deploy/pim-backend-api -f
```

### Verify DB connectivity
The entrypoint runs migrations / schema push automatically. Confirm tables exist:
```bash
kubectl -n crdb exec -it statefulset/cockroachdb -- \
	/cockroach/cockroach sql --insecure -e "\dt testdb.public.*" || true
```

### ArgoCD application
`pim-backend-application.yaml` deploys the backend from branch `pim-backend-api-config`.
```bash
kubectl apply -f pim-backend-application.yaml
kubectl -n argocd get application pim-backend-api -w
```

### Port-forward backend (optional)
```bash
kubectl -n pim port-forward svc/pim-backend-api 3000:3000
curl http://localhost:3000/api/health || true
```

### Next enhancements
* Move `DATABASE_URL` into a Secret.
* Add JWT / app secrets via Secret + `envFrom`.
* Use image tags tied to Git commit SHAs.
* Multi-node CockroachDB in secure mode.
* Dedicated migration Job instead of entrypoint migrations (for stricter control).

### Sealed Secrets (DATABASE_URL)

The backend's `DATABASE_URL` is now stored as a Bitnami SealedSecret for GitOps safety (only the cluster with the private key can decrypt it).

Files involved:
* `crdb-minikube/pim-backend/sealed-secret-db-url.yaml` (committed, encrypted value)
* Decrypted runtime Secret: `pim-backend-db-url` (namespace `pim`)
* Deployment consumes it via `env.secretKeyRef`.

Generate / rotate:
```bash
kubeseal --controller-name=sealed-secrets-controller \
				 --controller-namespace=kube-system \
				 --fetch-cert > sealed-secrets.crt

cat > /tmp/db-url-secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
	name: pim-backend-db-url
	namespace: pim
type: Opaque
stringData:
	DATABASE_URL: postgresql://root@cockroachdb-public.crdb.svc.cluster.local:26257/testdb?sslmode=disable
EOF

kubeseal --format=yaml --cert=sealed-secrets.crt < /tmp/db-url-secret.yaml > /tmp/sealed-secret-db-url.yaml
grep DATABASE_URL /tmp/sealed-secret-db-url.yaml   # copy encrypted value
```
Paste the encrypted value into `sealed-secret-db-url.yaml` under `spec.encryptedData.DATABASE_URL`, commit, push, and let ArgoCD sync.

Verification:
```bash
kubectl -n pim get sealedsecret pim-backend-db-url
kubectl -n pim get secret pim-backend-db-url -o jsonpath='{.data.DATABASE_URL}' | base64 -d
kubectl -n pim exec deploy/pim-backend-api -- /bin/sh -c 'echo $DATABASE_URL'
```

If you reinstall the sealed-secrets controller (losing its private key), you must reseal all SealedSecrets (old encrypted blobs will no longer decrypt).


## ArgoCD deployment

Apply the ArgoCD `Application` manifest to manage and continuously sync these manifests:

```
cockroachdb-application.yaml       # ArgoCD Application pointing to ./crdb-minikube
```

### Steps

```bash
# (Assumes ArgoCD installed; namespace argocd exists)
kubectl apply -f cockroachdb-application.yaml

# Watch Argo sync
kubectl -n argocd get applications cockroachdb-single-node -w

# After Healthy/Synced, verify database
kubectl -n crdb exec -it statefulset/cockroachdb -- \
	/cockroach/cockroach sql --insecure -e "SHOW DATABASES;"
```

### Customizing
* Change `targetRevision` to main or a tag as needed.
* Remove automated prune/selfHeal for manual approval flows.
* Multi-node: modify `cockroachdb-statefulset.yaml` to use `start` and add join flags; add a one-time init Job manifest.


## Cleanup

```bash
kubectl delete namespace crdb
minikube delete   # if you want to remove the whole cluster
```

## Next steps

* Convert to a 3-node StatefulSet.
* Add TLS cert generation job & secrets.
* Integrate into CI (apply then smoke test `SHOW DATABASES`).
* Parameterize version via Kustomize or Helm if needed later.

---

## End-to-End Validation Checklist

1. ArgoCD Applications
	```bash
	kubectl -n argocd get applications
	```
	Expect both `cockroachdb-single-node` and `pim-backend-api` to be `Synced` & `Healthy`.

2. CockroachDB pod & Job
	```bash
	kubectl -n crdb get pods
	kubectl -n crdb logs job/bootstrap-sql
	```
	Expect `bootstrap-sql` status `Completed` and `cockroachdb-0` Ready.

3. Database presence
	```bash
	kubectl -n crdb exec cockroachdb-0 -- \
	  /cockroach/cockroach sql --insecure -e "SHOW DATABASES;" | grep testdb
	```

4. Backend pod health
	```bash
	kubectl -n pim get pods -l app=pim-backend-api
	kubectl -n pim logs deploy/pim-backend-api | grep -i prisma || true
	```

5. (Optional) List tables created by Prisma
	```bash
	kubectl -n crdb exec cockroachdb-0 -- \
	  /cockroach/cockroach sql --insecure -e "SHOW TABLES FROM testdb.public;"
	```

6. Backend health endpoint
	```bash
	kubectl -n pim port-forward svc/pim-backend-api 3000:3000 &
	curl -sf http://localhost:3000/api/health || echo "health check failed"
	```

## Troubleshooting

| Symptom | Likely Cause | Remedy |
|---------|--------------|--------|
| Backend pod CrashLoop (P1001) | CockroachDB not ready or `testdb` absent | Ensure `cockroachdb-0` Ready; re-apply kustomize to re-run bootstrap job |
| Bootstrap Job stuck `ContainerCreating` | ConfigMap not in `crdb` namespace | Add `namespace: crdb` to kustomization and re-apply |
| ArgoCD app Healthy but no resources | Path or targetRevision mismatch / repo private | Confirm repo URL, branch, and path; check ArgoCD repo permissions |
| Port-forward conflict on 8080 | Multiple forwards using same local port | Use alternative local port: `8081:8080` |
| Tables missing after startup | Prisma migrations not executed yet | Check logs, ensure entrypoint runs migrate or add migration Job |

## Enhancement Roadmap

Short-term (low friction):
* Move `DATABASE_URL` to a Secret (then remove it from ConfigMap).
* Add HTTP readiness probe for CockroachDB (`/health?ready=1`).
* Tag backend images with Git commit SHAs using Kustomize `images:` block.
* Add simple smoke test script (CI) executing `SHOW DATABASES;` after sync.

Mid-term:
* Dedicated migration Job (Prisma `migrate deploy`) instead of doing it in the app container.
* Multi-node CockroachDB (3 replicas, `start` + `--join` flags, one-time init job).
* TLS (cert generation + Secrets, replace `--insecure`).
* Observability: Prometheus scraping + Grafana dashboards, structured logs.

Long-term:
* App-of-Apps ArgoCD pattern (one root Application manages both layers).
* Automated image build & promotion pipeline (dev -> staging -> prod branches).
* Rollbacks via ArgoCD sync to previous revisions + schema drift detection.

## Why Separate DB Creation and Schema Migrations?

* Logical DB creation (`CREATE DATABASE IF NOT EXISTS testdb`) is stable infra state, easy to express declaratively and idempotently.
* Table/column migrations evolve with application releases—keeping them with the app (or a migration Job tied to app version) ensures schema matches code.
* This separation reduces cross-team coupling and makes rollback clearer (roll back app image & associated migration Job; the database existence remains intact).

---

## Glossary

| Term | Meaning |
|------|---------|
| Logical Database | Named database inside CockroachDB (`testdb`) distinct from the cluster itself |
| Bootstrap | One-time (idempotent) creation of base logical DB objects via Job |
| Kustomize | Overlay tool assembling manifests and generating ConfigMaps/Secrets |
| ArgoCD Application | CRD that watches a Git path and syncs manifests into the cluster |

---

## License

This repository is for internal demo / experimentation. Add license terms here if distributing externally.

