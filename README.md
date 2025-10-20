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

