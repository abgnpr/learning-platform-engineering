# 3. Workloads, labels, and rollouts

[Index](README.md) · Previous: [Architecture](02-kubernetes-architecture.md) · Next: [Networking](04-services-and-networking.md)

## Remember these four things

- **Deployment → ReplicaSet → Pods = rollout → count → execution.**
- **Labels identify; selectors match. Selection is separate from ownership.**
- **A template change rolls out; a replica-count change scales.**
- **Undo restores a Pod template, not database contents.**

## Two relationships to keep separate

**Goal:** explain why deleting one execution unit differs from changing desired state. Prerequisites: a Ready Minikube node and the [architecture](02-kubernetes-architecture.md) chapter. Run the lab below from the repository root.

Why have three kinds? A Pod describes an execution unit. A ReplicaSet maintains a count from a particular Pod template. A Deployment coordinates ReplicaSets as the template changes, allowing gradual replacement and revision history. Keeping only a bare Pod gives you no higher-level controller to create another Pod after deletion. A kubelet may restart a container inside that existing Pod; that is a different operation.

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

## Choose a controller from the work's lifetime

| Need | Usual starting point | Reason |
| --- | --- | --- |
| Interchangeable long-running web replicas | Deployment | Revision management and replaceable replicas |
| Stable per-replica identity/storage | StatefulSet | Ordered identities and claim relationships; app still handles data coordination |
| Run a task to completion | Job | Tracks completion and retries rather than keeping a server running forever |
| Start Jobs on a schedule | CronJob | Creates scheduled Jobs; tasks should tolerate retries/overlap according to policy |
| A node-local agent on eligible nodes | DaemonSet | Places a Pod on each selected node |
| Temporary observation/debugging | Standalone Pod | Simple execution; no controller to replace a deleted Pod |

Do not put every task in a Deployment. A completed migration Job and a continuously serving API have different success conditions. The [YAML manual](10-kubernetes-yaml-manual.md) provides shapes for these kinds; database caveats are in [storage](13-storage-and-state.md).

## Labels identify; selectors match

| YAML location | Meaning |
| --- | --- |
| `metadata.labels` | Labels this particular object |
| Deployment `spec.selector.matchLabels` | Selector used to identify its workload; must match template labels |
| `spec.template.metadata.labels` | Labels assigned to newly created Pods |
| Service `spec.selector` | Labels used to select Pods in the Service's namespace |

`app=nginx` is an application label. `pod-template-hash` distinguishes Deployment-managed templates/ReplicaSets; it is not an image digest. Ownership is also recorded through owner references, not inferred just from a shared label.

```bash
kubectl get pods -n foundation --show-labels
kubectl get pods -n foundation -l app=nginx    # Filter by label selector
kubectl get deployment,replicaset,pod -n foundation
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
  namespace: foundation
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

Read it aloud: **“In foundation, keep three Pods from this template; label them app=nginx; run a container named nginx using nginx:alpine.”** The template is the blueprint for replacements as well as initial Pods.

| Part | Question it answers |
| --- | --- |
| `apiVersion`, `kind` | What API object is this? |
| `metadata` | What is it called, where does it belong, and how is it labelled? |
| `spec` | What state do I want? |
| `status` in a live response | What state does Kubernetes currently observe? |

For saved manifests, keep intended configuration; remove generated bookkeeping such as UID, resource version, timestamps, and status. This minimal example omits resource settings and probes.


</details>

## Create, inspect, and scale

Assume namespace `foundation` exists. The create command is for a Deployment that does not already exist.

```bash
kubectl create deployment nginx -n foundation --image=nginx:alpine
kubectl scale deployment nginx -n foundation --replicas=3
kubectl get deployment nginx -n foundation
kubectl get replicasets -n foundation
kubectl describe deployment nginx -n foundation
kubectl get deployment nginx -n foundation -o yaml
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
kubectl set image deployment/nginx -n foundation nginx=nginx:latest
kubectl get pods -n foundation -w          # Watch; Ctrl+C stops the watch only
kubectl rollout status deployment/nginx -n foundation --timeout=120s
kubectl get replicasets -n foundation
kubectl rollout history deployment/nginx -n foundation
kubectl rollout history deployment/nginx -n foundation --revision=1
```

The two image tags above are a syntax example, not proof of a meaningful software change: tags can move and may resolve to the same bytes. The lab below uses an intentionally missing image to make failed rollout behavior observable.

Default rolling-update values are 25% `maxUnavailable` and 25% `maxSurge`. With three desired replicas, rounding permits **zero unavailable** and **one extra** Pod. Extra capacity helps start new Pods before removing old ones. Terminating Pods can temporarily complicate total counts. See [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

## Rollback restores an earlier Pod template

```bash
kubectl rollout undo deployment/nginx -n foundation
kubectl rollout status deployment/nginx -n foundation --timeout=120s
kubectl get deployment nginx -n foundation \
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

## Lab: watch reconciliation and recover a failed revision

Reserve namespace `foundation` for the guided exercises. If it already contains other work, choose a new namespace and update the lab files before proceeding. The following applies the base stage; after chapter 12 use its configuration stage for recovery instead.

**Predict:** how many Deployment, ReplicaSet, and Pod objects will exist? Which object stores the desired replica count?

```bash
kubectl config current-context
kubectl apply -f labs/kubernetes/namespace.yaml
kubectl apply -f labs/kubernetes/deployment.yaml
kubectl rollout status deployment/nginx -n foundation --timeout=180s
kubectl get deployment,replicaset,pod -n foundation -o wide
kubectl get pods -n foundation -l app=nginx \
  -o 'custom-columns=NAME:.metadata.name,UID:.metadata.uid,OWNER:.metadata.ownerReferences[0].name,IP:.status.podIP'
```

Expect one Deployment, initially one ReplicaSet, and three ready Pods. Rerunning after rollouts can show retained older ReplicaSets. Choose one actual Pod name and use it in place of `POD_NAME`:

```bash
kubectl describe pod POD_NAME -n foundation
kubectl delete pod POD_NAME -n foundation
kubectl rollout status deployment/nginx -n foundation --timeout=180s
kubectl get pods -n foundation -l app=nginx -o wide
```

**Observe:** one replacement Pod has a new name/UID. Its IP may change; IP reuse is possible, so the UID is the stronger identity evidence. Desired replicas remains three. Then scale to two and back to three, inspecting between changes. Explain why this changes the count without creating an image revision.

**Introduce a failed revision:** the following tag is deliberately invalid. Before running it, predict whether old Pods must immediately disappear.

```bash
kubectl set image deployment/nginx -n foundation nginx=nginx:foundation-missing-image
kubectl rollout status deployment/nginx -n foundation --timeout=60s
kubectl get pods,replicasets -n foundation
```

The status command is expected to time out/nonzero. Inspect the **new failing Pod** with `describe`; find the image-pull event. With three healthy old replicas and default rolling-update settings, old Pods can keep serving while the new Pod cannot start. A default Deployment reports a stalled rollout; it does not automatically undo it for you.

**Recover:**

```bash
kubectl rollout undo deployment/nginx -n foundation
kubectl rollout status deployment/nginx -n foundation --timeout=180s
kubectl get deployment nginx -n foundation \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expect the previous image reference and three available replicas. If revision history has been removed or you made other intervening revisions, reapply the latest appropriate lab stage instead. Inspect history before choosing a revision.

**Checkpoint:** explain who recreated the deleted Pod, who ran its containers, why the failed image stalled, and why undo restores a template rather than data. Keep the healthy Deployment for networking. If the initial healthy rollout fails too, investigate its actual events; that is a prerequisite failure rather than the intended invalid-tag experiment.

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
