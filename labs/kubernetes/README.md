# Kubernetes lab files

[Learning route](../../README.md)

Run commands from the repository root on a local Minikube cluster. These fixtures accompany the lessons; do not apply this whole directory. Two files intentionally define successive versions of the same Deployment.

| Stage | Files | Lesson |
| --- | --- | --- |
| Scope | `namespace.yaml` | Apply first; creates `foundation` |
| Execution | `deployment.yaml` | Chapter 3; three nginx Pods, initially without probes/resources |
| Network | `service.yaml` | Chapter 4; Service port **8080**, named backend port `http` = **80** |
| Configuration | `configuration.yaml` | Chapter 12; ConfigMap plus replacement Deployment definition with probes/resources/mount |
| Probe experiment | `health-demo.yaml` | Chapter 12; nginx Pod with controllable marker checks and its own Service |
| API permissions | `access.yaml` | Chapter 14; ServiceAccount, Role, RoleBinding |
| Process restrictions | `security-demo.yaml` | Chapter 14; non-root Pod with read-only root filesystem and writable `/tmp` |
| Scheduling failure | `unschedulable.yaml` | Chapter 6; intentionally excessive CPU request; delete after observing |
| Process failure | `crash-demo.yaml` | Chapter 7; deliberate exit code 1 and repeated restarts |

`kubectl apply -f` stores intent; `kubectl rollout status` waits for a Deployment's progress. When a command times out, inspect conditions, events, and logs before choosing a recovery.

## Record your environment

```bash
minikube version
kubectl version
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'
kubectl get pods -n foundation -l app=nginx \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.uid}{" "}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

The last command applies after nginx starts. Record image IDs because tags can move; different nodes can cache different image contents. For a fully repeatable run, resolve and use tested registry digests in **both** Deployment stages, and record the cluster/tool versions. Do not invent a digest or treat a container-runtime-specific image ID as automatically usable as a registry reference. See [image names and digests](https://kubernetes.io/docs/concepts/containers/images/).

## Recovery and reset

For chapters 3–6, reapply `deployment.yaml` and `service.yaml` as applicable. After chapter 12, recover with `configuration.yaml` and `service.yaml`; applying the base Deployment can remove configuration/probes/resources. Configuration edits made with `kubectl patch` are live experiments: the saved file still contains the original values.

A named standalone Pod can be inspected and reapplied unchanged; changes to most of its spec require deleting that exercise Pod and recreating it. `security-demo` sleeps for an hour and can restart normally afterward. Use events and timing to connect a restart to its cause.

Optional final teardown, **only when you want all exercise resources removed**:

```bash
kubectl delete namespace foundation
```

This does not delete `db-lab` or the independent challenge. Use `minikube stop` to retain your cluster for another session.

## Validation boundary

Authoring checks on 2026-11-01 covered local links/anchors, Bash/Zsh syntax, duplicate YAML keys, Kubernetes 1.37 schema validation, manifest reference relationships, and strict server dry-run acceptance. Server dry-runs used temporary object names and the existing `default` namespace for namespaced resources; no objects were persisted.

The full runtime learning route has **not** been executed end to end as part of this edit. Schema/dry-run checks do not establish image pull success, application responses, configuration projection timing, probe behavior, storage provisioning, or backup restoration. Treat each lesson's required evidence as the runtime acceptance checklist; record actual versions and results when completing it.
