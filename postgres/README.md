# PostgreSQL lab: persistence and a separate recovery copy

[Index](../README.md) · Companion: [Storage and state](../13-storage-and-state.md)

**Goal:** prove that a database row survives Pod replacement, then restore a logical backup into another database. Prerequisites: Ready Minikube cluster, storage/configuration concepts, and image-registry access. Run **from the repository root**, not this subdirectory. No host PostgreSQL installation is needed: the teaching image includes `psql`, `pg_dump`, and `pg_restore`.

Use a new **`db-lab`** namespace reserved for this exercise. The manifests explicitly name it; older copies of this example in `default` are separate resources. Do not apply these files to migrate an existing database. The password in [secret.yaml](secret.yaml) is a public teaching value, not a real credential.

## 1. Inspect the design and storage prerequisites

```text
Secret key → POSTGRES_PASSWORD during first initialization
PVC postgres-pvc → volume postgres-storage → /var/lib/postgresql/data
Deployment (one replica, Recreate) → PostgreSQL Pod
ClusterIP Service → app=postgres → named backend port postgres (5432)
```

Read [deployment.yaml](deployment.yaml), [storage.yaml](storage.yaml), and [service.yaml](service.yaml). The workload uses PostgreSQL major version 15; do not change a database major version by simply replacing an image tag. Recreate intentionally permits downtime during a template upgrade and avoids the ordinary rolling-update surge. It is a teaching choice, not database high availability or full fencing. Keep replicas at one.

```bash
kubectl config current-context
kubectl get storageclass
minikube addons list
kubectl create namespace db-lab --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f postgres/secret.yaml -f postgres/storage.yaml
kubectl apply -f postgres/deployment.yaml -f postgres/service.yaml
kubectl rollout status deployment/postgres -n db-lab --timeout=240s
kubectl get pvc,pod,svc -n db-lab
kubectl describe pvc postgres-pvc -n db-lab
```

Expect a Bound PVC and one ready Pod. A class with `WaitForFirstConsumer` may wait until the Pod is created; do not require binding before creating its consumer. Standard Minikube storage labs normally use the `storage-provisioner` and `default-storageclass` addons. If absent/disabled, enable those addons deliberately on your local cluster with `minikube addons enable storage-provisioner` and `minikube addons enable default-storageclass`, then inspect claim/Pod events again. A non-Minikube cluster needs its own supported provisioner/class.

If the rollout times out, inspect Pod events and logs plus PVC events. Distinguish image-pull failure, unbound storage, and database startup failure. `pg_isready` probes check server availability; they do not prove data correctness, successful authentication, or application-level transactions.

## 2. Prove the Service path and write a row

Start forwarding in another host terminal:

```bash
kubectl port-forward -n db-lab svc/postgres-service 15432:5432
```

This provides `127.0.0.1:15432` for a host database client if you have one. PostgreSQL uses its own protocol; a browser or HTTP `curl` cannot test a SQL transaction. Port-forward selects a Pod and bypasses the ordinary Service VIP path. Stop it with Ctrl+C when finished.

To test actual in-cluster Service DNS/TCP/authentication, execute the following command. It prompts for the public lab password `postgres`; no secret value appears in the command arguments:

```bash
kubectl exec -it -n db-lab deployment/postgres -- \
  psql -h postgres-service -U postgres -d postgres -W
```

At the **psql prompt**, run SQL:

```sql
CREATE DATABASE foundation;
\connect foundation
CREATE TABLE evidence (id integer PRIMARY KEY, note text NOT NULL);
INSERT INTO evidence VALUES (1, 'survives Pod replacement');
SELECT * FROM evidence;
\quit
```

Expect one row with ID 1. This initialization assumes a fresh lab database; if it already exists from an earlier run, inspect and reuse it rather than recreating or dropping it blindly. `\connect` and `\quit` are psql commands, not host shell commands.

## 3. Replace execution without deleting storage

**Predict:** which UID changes, and which claim/PV relationship should stay the same?

```bash
kubectl get pods -n db-lab -l app=postgres \
  -o custom-columns=NAME:.metadata.name,UID:.metadata.uid
kubectl get pvc postgres-pvc -n db-lab -o wide
```

Record the actual Pod name, UID, and bound volume. Substitute that name for `POD_NAME`:

```bash
kubectl delete pod POD_NAME -n db-lab
kubectl rollout status deployment/postgres -n db-lab --timeout=240s
kubectl get pods -n db-lab -l app=postgres \
  -o custom-columns=NAME:.metadata.name,UID:.metadata.uid
kubectl get pvc postgres-pvc -n db-lab -o wide
kubectl exec -n db-lab deployment/postgres -- \
  psql -U postgres -d foundation -c 'SELECT * FROM evidence;'
```

The last command uses the container's local Unix socket under the teaching image's local-auth configuration; it is a different path from the earlier password-authenticated Service connection. Expect a new Pod UID, the same PVC/PV binding, and the original row. Do not force-delete the database Pod: graceful shutdown and single-writer behavior matter.

## 4. Make a backup and restore it elsewhere

On the **host**, make a private temporary directory and write a custom-format dump there. Keep this shell open so `DB_LAB_BACKUP_DIR` remains defined:

```bash
DB_LAB_BACKUP_DIR=$(mktemp -d /tmp/k8s-db-backup.XXXXXX)
kubectl exec -n db-lab deployment/postgres -- \
  pg_dump -U postgres -d foundation -Fc > "$DB_LAB_BACKUP_DIR/foundation.dump"
test -s "$DB_LAB_BACKUP_DIR/foundation.dump"
kubectl exec -n db-lab deployment/postgres -- \
  createdb -U postgres foundation_restore
kubectl exec -i -n db-lab deployment/postgres -- \
  pg_restore -U postgres -d foundation_restore --exit-on-error \
  < "$DB_LAB_BACKUP_DIR/foundation.dump"
kubectl exec -n db-lab deployment/postgres -- \
  psql -U postgres -d foundation_restore -c 'SELECT * FROM evidence;'
```

Continue only if each command succeeds. Expect the same row in **`foundation_restore`**, a separately created database. Do not use `-t` for dump/restore: terminal processing can corrupt binary archive data. The restore database name must be unused; for a repeated drill choose a fresh name and update both restore/check commands.

This proves a logical dump is readable and restorable on the same server version. It does **not** prove full node-loss/disaster recovery, role/global-object recovery, an off-host backup schedule, or recovery-time objectives. The dump sits on your host in `/tmp`, outside the database Pod but still on the laptop; copy real backups to durable storage in a separate failure domain. `pg_dump` covers a database, not all server globals. See [pg_dump](https://www.postgresql.org/docs/15/app-pgdump.html) and [pg_restore](https://www.postgresql.org/docs/15/app-pgrestore.html).

## Password and upgrade traps

The image uses `POSTGRES_PASSWORD` to initialize an **empty** data directory. Editing the Secret after initialization does not automatically change the existing PostgreSQL role password. A controlled password change must coordinate database SQL, credential storage, and clients. Never delete the PVC merely to make a new password take effect.

Similarly, `rollout undo` restores a Kubernetes Pod template; it does not reverse SQL changes or restore data. The image version and its data-directory rules matter. See the [official image documentation](https://hub.docker.com/_/postgres).

## Finish and explain

Show the original row after replacement and the row in the restored database. Explain UID versus PVC lifetime, the difference between socket and Service tests, why a PVC is not a backup, and why multiple replicas do not implement PostgreSQL replication.

To retain the exercise, use `minikube stop`. Optional destructive teardown is `kubectl delete namespace db-lab`: this deletes the PVC and, depending on the PV reclaim policy/provisioner, can delete the backing data. Inspect `kubectl get pv` first. The dump remains in the printed temporary path, which you can display using `printf '%s\n' "$DB_LAB_BACKUP_DIR"`; remove it only when you no longer need the exercise backup.
