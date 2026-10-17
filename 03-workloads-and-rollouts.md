# 3. Workloads, labels, and rollouts

[Index](README.md) · Previous: [Architecture](02-kubernetes-architecture.md) · Next: [Networking](04-services-and-networking.md)

## Remember these four things

- **Deployment → ReplicaSet → Pods = rollout → count → execution.**
- **Labels identify; selectors match. Selection is separate from ownership.**
- **A template change rolls out; a replica-count change scales.**
- **Undo restores a Pod template, not database contents.**

## Two relationships to keep separate

```text
Ownership:   Deployment → ReplicaSet → Pods
Selection:   Service ── matches Pod labels ──→ backend Pods
```

The Service routes traffic to selected Pods. It does not own them or create replacements.

| Object | What you declare | What it handles |
| --- | --- | --- |
| Deployment | Pod template, replicas, update strategy | ReplicaSets and rollout progression |
| ReplicaSet | Desired count of matching Pods | Creating/replacing missing Pods |
| Pod | Containers and their configuration | The application execution unit |

Manage long-running app intent through a Deployment. Deleting a managed Pod usually causes replacement; scaling changes the desired count.

## Labels identify; selectors match

| YAML location | Meaning |
| --- | --- |
| `metadata.labels` | Labels this particular object |
| Deployment `spec.selector.matchLabels` | Selector used to identify its workload; must match template labels |
| `spec.template.metadata.labels` | Labels assigned to newly created Pods |
| Service `spec.selector` | Labels used to select Pods in the Service's namespace |

`app=nginx` is an application label. `pod-template-hash` distinguishes Deployment-managed templates/ReplicaSets; it is not an image digest. Ownership is also recorded through owner references, not inferred just from a shared label.

```bash
kubectl get pods -n demo --show-labels
kubectl get pods -n demo -l app=nginx    # Filter by label selector
kubectl get deployment,replicaset,pod -n demo
```

## Read YAML as intent

For authoring from scratch, use the [YAML writing manual](10-kubernetes-yaml-manual.md). It covers the common object kinds, field locations, and matching references.

<details>
<summary>Open a minimal Deployment YAML and learn to read it</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
```

Read it aloud: **“In demo, keep three Pods from this template; label them app=nginx; run a container named nginx using nginx:alpine.”** The template is the blueprint for replacements as well as initial Pods.

| Part | Question it answers |
| --- | --- |
| `apiVersion`, `kind` | What API object is this? |
| `metadata` | What is it called, where does it belong, and how is it labelled? |
| `spec` | What state do I want? |
| `status` in a live response | What state does Kubernetes currently observe? |

For saved manifests, keep intended configuration; remove generated bookkeeping such as UID, resource version, timestamps, and status. This minimal example omits resource settings and probes.


</details>

## Create, inspect, and scale

Assume namespace `demo` exists. The create command is for a Deployment that does not already exist.

```bash
kubectl create deployment nginx -n demo --image=nginx:alpine
kubectl scale deployment nginx -n demo --replicas=3
kubectl get deployment nginx -n demo
kubectl get replicasets -n demo
kubectl describe deployment nginx -n demo
kubectl get deployment nginx -n demo -o yaml
```

For a saved manifest, `kubectl apply -f FILE.yaml` creates or updates declared intent. Keep that manifest consistent after imperative changes so a later apply does not restore old configuration.

In ReplicaSet output, **desired** is the requested count, **current** is the existing count, and **ready** is the ready count. These values can differ while the system is progressing or failing.

## An update changes the Pod template

```text
Template A → ReplicaSet A → old Pods
                         ↓ gradual replacement
Template B → ReplicaSet B → new Pods
```

Changing an image creates a template revision. Changing replicas alone does not.

```bash
# The left nginx is the container name; the right side is its image
kubectl set image deployment/nginx -n demo nginx=nginx:latest
kubectl get pods -n demo -w          # Watch; Ctrl+C stops the watch only
kubectl rollout status deployment/nginx -n demo --timeout=120s
kubectl get replicasets -n demo
kubectl rollout history deployment/nginx -n demo
kubectl rollout history deployment/nginx -n demo --revision=1
```

Default rolling-update values are 25% `maxUnavailable` and 25% `maxSurge`. With three desired replicas, rounding permits **zero unavailable** and **one extra** Pod. Extra capacity helps start new Pods before removing old ones. Terminating Pods can temporarily complicate total counts. See [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

## Rollback restores an earlier Pod template

```bash
kubectl rollout undo deployment/nginx -n demo
kubectl rollout status deployment/nginx -n demo --timeout=120s
kubectl get deployment nginx -n demo \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Old ReplicaSets can be retained at zero replicas for revision history. Undo makes an earlier template current again; it does not undo database data changes or external side effects.

## Details when you need them

<details>
<summary>Rollout guarantees, image tags, pull policy, and container ports</summary>

| Confusion | Correct idea |
| --- | --- |
| Rollout complete means zero downtime | It means Kubernetes rollout conditions passed; application continuity needs separate checks |
| An old image tag guarantees old bytes | Moving tags can resolve differently; deliberate versions/digests improve reproducibility |
| Updating the tag resets pull policy | A stored pull policy is not automatically recalculated on a tag change |
| `containerPort: 80` opens the app | The process must listen; the field alone neither starts a listener nor exposes a node port |
| Delete a Pod to reduce replicas | Change the Deployment replica count |

See [images and pull policies](https://kubernetes.io/docs/concepts/containers/images/).

</details>

## Recall before checking

1. Which YAML section changes the blueprint for future Pods?
2. Why can four Pods appear during an update when replicas is three?
3. Why does the Service survive an image update?
4. Does undo restore database contents?

<details>
<summary>Answers</summary>

1. `spec.template`.
2. Rolling updates permit surge capacity; 25% of three rounds up to one extra Pod.
3. Its identity stays the same and its selector can continue matching replacement Pods.
4. No. It restores a previous Pod template.

</details>
