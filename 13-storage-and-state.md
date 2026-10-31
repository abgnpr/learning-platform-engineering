# 13. Storage and state: what survives replacement?

[Index](README.md) · Learning route: [Configuration and health](12-configuration-and-health.md) → this lesson → [Security and access](14-security-and-access.md)

**Goal:** predict data lifetime, trace storage references, and distinguish persistence from recovery. Prerequisites: Pod identity/replacement, namespaces, and configuration references. The hands-on work is the [PostgreSQL lab](postgres/README.md); it uses a separate namespace and keeps the HTTP application available.

## Start with the lifetime of the data

| Storage | Survives a container restart in the same Pod? | Survives Pod replacement? | Boundary |
| --- | --- | --- | --- |
| Container writable layer | Do not rely on it; a restarted container gets a fresh writable layer | No | Temporary container-local changes |
| `emptyDir` | Yes | No | Lifetime of that Pod; useful for scratch/shared work |
| PVC-backed volume | Normally, if the same claim/backing storage remains | Normally, with the same claim and usable backing storage | Depends on backend, attachment, deletion, and reclaim behavior |
| Off-node backup | Separate recovery copy | Can restore independently of the original Pod | Only useful if complete and a restore has been tested |

Podman stop/start of the *same container* in lesson 0 is different from Kubernetes restarting a container by creating a new container instance. A restart count does not mean a new Pod, and a stable Pod name does not turn a container writable layer into durable storage.

Memory-backed `emptyDir` consumes memory rather than durable disk. A normal disk-backed `emptyDir` still disappears when its Pod is removed. See [volumes](https://kubernetes.io/docs/concepts/storage/volumes/).

## Follow the references

```text
Deployment template
  volumes[].persistentVolumeClaim.claimName → namespaced PVC
  container.volumeMounts[].name             → volumes[].name
  container.volumeMounts[].mountPath        → path the application uses

PVC request → existing matching PV or dynamic provisioner via StorageClass
            → bound PV → real storage backend
```

The **PVC** requests storage. The **PV** represents provisioned backing storage and is cluster-scoped. A **StorageClass** describes provisioning policy. A dynamic provisioner can create storage for a claim; a YAML claim alone cannot manufacture a working storage backend.

The scheduler, volume binding, and provisioning can interact. A claim may remain Pending until a consumer can be scheduled when its class uses `WaitForFirstConsumer`. A missing/default class, unavailable provisioner, or incompatible access mode/size can also block progress. Read events rather than assuming every Pending PVC is broken for the same reason.

```bash
kubectl get storageclass
kubectl get pvc -n db-lab
kubectl describe pvc postgres-pvc -n db-lab
kubectl get pv
```

The claim commands become applicable after creating the database lab. Inspect the bound PV's reclaim policy. With `Delete`, deleting a claim may delete backing storage; with `Retain`, storage remains but needs administrative recovery/reuse steps. Local Minikube storage is tied to local infrastructure; deleting a profile can destroy the data. [Persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

## Access mode is not a database concurrency policy

`ReadWriteOnce` means writable mounting by one **node**, not necessarily one Pod. Several Pods on that node may access the volume. It is not a database lock or a guarantee of one writer.

The teaching PostgreSQL Deployment therefore uses one replica and **Recreate** for deliberate template upgrades: the old revision is terminated before the new one starts. This causes downtime and is a simple local learning choice. Do not scale it to multiple independent writers. Recreate does not provide a general fencing guarantee during all failure/deletion scenarios.

A **StatefulSet** provides stable ordinal identities and storage relationships for its Pods. It is useful when those identities matter; it does not itself implement database replication, backups, leader election, or safe major-version upgrades. Database operators add application-specific management, with their own operational requirements. See [Deployment strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) and [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/).

## Lab: persist a row, then prove recovery separately

Complete the [PostgreSQL walkthrough](postgres/README.md). Before starting, write predictions for these events:

1. Replacing the PostgreSQL Pod while preserving its PVC.
2. Editing the password Secret after the database has already initialized.
3. Restoring a logical dump into a second database.
4. Deleting the namespace that owns the PVC.

The lab exercises the first three. The fourth is explained rather than required because it intentionally removes the claim and may destroy data. Never use namespace deletion as an ordinary database restart.

**Required evidence:** a row before/after Pod replacement; different Pod UID; the same claim/PV relationship; a successful connection through the Service; and the row in a separately restored database. A healthy Pod alone proves none of these data properties.

## Checkpoint

Explain why a PVC, a StatefulSet, three replicas, and a backup each solve different problems. Draw the claim/mount references without looking. If a database Pod is Pending, choose checks that distinguish scheduling failure from unbound storage. If it runs but data is missing, check the database and mounted path/claim before assuming Kubernetes deleted the data.

Continue to [security and access](14-security-and-access.md).
