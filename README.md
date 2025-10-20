# test-initdb

Minimal CockroachDB deployment on Minikube using a SQL "initdb" style bootstrap to create a `testdb` database.

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

