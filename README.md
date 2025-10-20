# test-initdb

Minimal CockroachDB deployment on Minikube using a SQL "initdb" style bootstrap to create a `testdb` database.

## Contents

`crdb-minikube/` holds a tiny, ordered set of Kubernetes manifests plus an `init.sql` file:

```
crdb-minikube/
	00-namespace.yaml                # Creates dedicated namespace `crdb`
	10-services.yaml                 # Headless service for pod DNS + public service for client/UI access
	20-statefulset.yaml              # Single-node CockroachDB in insecure mode (auto-initializes)
	40-bootstrap-sql-configmap.yaml  # Stores your SQL bootstrap (idempotent)
	50-bootstrap-sql-job.yaml        # Executes the SQL against the running node
	init.sql                         # Actual SQL: creates `testdb`
```

## Quick start

```bash
minikube start --cpus=4 --memory=6144

# Apply base objects
kubectl apply -f crdb-minikube/00-namespace.yaml
kubectl apply -f crdb-minikube/10-services.yaml
kubectl apply -f crdb-minikube/20-statefulset.yaml

# Wait for CockroachDB pod readiness
kubectl -n crdb rollout status statefulset/cockroachdb

# Initialize the cluster (system ranges)
# (Single-node mode auto-initializes; cluster init job removed)

# Create the testdb database
kubectl apply -f crdb-minikube/40-bootstrap-sql-configmap.yaml
kubectl apply -f crdb-minikube/50-bootstrap-sql-job.yaml

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
* SQL bootstrap can include users/roles/grants; keep statements idempotent (`IF NOT EXISTS`) to allow safe reapply.

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

