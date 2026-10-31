# 9. Command sheet and practice

[Index](README.md) · Previous: [Recall questions](08-interview-preparation.md) · Next: [YAML manual](10-kubernetes-yaml-manual.md)

## Read kubectl commands once, then recognize the pattern

```text
kubectl VERB RESOURCE NAME -n NAMESPACE OPTIONS
        │       │      │        │
        action  kind   object   scope
```

For example, `kubectl describe pod nginx-abc -n foundation` means **describe → this Pod → in foundation**.

| Option | Meaning |
| --- | --- |
| `-n foundation` | Target namespace foundation |
| `-A` | All namespaces for supported listing commands |
| `-o wide` | More columns, including Pod IP and node |
| `-o yaml` | Full object representation |
| `-l app=nginx` | Match labels |
| `-w` | Watch changes; Ctrl+C stops watching |
| `-f FILE.yaml` | Read an object definition from a file |
| `--dry-run=client -o yaml` | Generate a manifest without submitting the create |

Short names you will recognize: `po` = Pods, `deploy` = Deployments, `rs` = ReplicaSets, `svc` = Services, `ns` = namespaces.

## Command sheet

Need to write the manifest rather than find a command? Use the [YAML templates and field manual](10-kubernetes-yaml-manual.md).

Examples target namespace `foundation` and app/container `nginx`. Replace uppercase placeholders. Create/expose commands require their names to be unused; inspect first.

These are command references, not a script to execute from top to bottom. The guided workload already exists if you followed the route. Its Service uses port **8080**; the shorter revision exercise later deliberately creates its own port-80 Service in another namespace.

### Orient and inspect

```bash
minikube start
minikube status
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,svc -A
kubectl get deployment,replicaset,pod,service -n foundation
kubectl get pods -n foundation -o wide --show-labels
kubectl describe pod POD_NAME -n foundation  # Object state and events
kubectl logs POD_NAME -n foundation          # Current application output
kubectl logs POD_NAME -n foundation --previous # Previous container instance
```

### Declare and change intent

```bash
kubectl create namespace foundation
kubectl create deployment nginx -n foundation --image=nginx:alpine
kubectl apply -f FILE.yaml
kubectl scale deployment nginx -n foundation --replicas=3
kubectl set image deployment/nginx -n foundation nginx=nginx:latest
kubectl rollout status deployment/nginx -n foundation --timeout=120s
kubectl rollout history deployment/nginx -n foundation
kubectl rollout undo deployment/nginx -n foundation
```

### Connect and debug

```bash
kubectl expose deployment nginx -n foundation --port=8080 --target-port=80
kubectl describe svc nginx -n foundation
kubectl get endpointslices -n foundation -l kubernetes.io/service-name=nginx -o yaml
kubectl port-forward -n foundation svc/nginx 8080:8080 # Local 8080 → Service 8080 → Pod 80
```

For a NodePort Service, create it with `--type=NodePort`; use `minikube service nginx -n foundation --url` to find the local access URL. For DNS/application test Pods, see [the three test commands](05-dns-and-namespaces.md#test-from-inside-the-cluster).

### Stop versus delete

```bash
minikube stop                         # Retain cluster state
kubectl delete pod POD_NAME -n foundation   # Managed Pod can be replaced automatically
```

`kubectl delete namespace NAME` deletes resources in that namespace. `minikube delete -p PROFILE` destroys the local profile. Use these only for intentional teardown.

## Practise predicting, not just typing

Use `revision-lab` for this exercise so existing `foundation` resources can stay separate. Confirm `kubectl config current-context` is your local Minikube context. Exercise commands change the cluster and are not run by reading these notes.

1. Create a namespace, an nginx Deployment with three replicas, and a ClusterIP Service.
2. Before inspecting, predict the ownership chain and Service selector. Verify them.
3. From a Pod in another namespace, call `nginx.revision-lab`. Predict what bare `nginx` would look for.
4. Delete one exercise nginx Pod. Predict count, ownership, and which identity/address can change.
5. Change the image, inspect ReplicaSets/history, then undo. Predict what the Service retains.
6. Explain one output aloud without notes. If your explanation uses “Kubernetes just does it,” identify the component or selector involved.

<details>
<summary>Reveal a worked command sequence</summary>

Assumes the namespace name is yours to use. Dry-run plus apply lets you regenerate the intended setup. Moving tags are convenient learning examples; tested versions/digests provide better reproducibility.

```bash
kubectl create namespace revision-lab --dry-run=client -o yaml | kubectl apply -f -
kubectl create deployment nginx -n revision-lab --image=nginx:alpine \
  --replicas=3 --dry-run=client -o yaml | kubectl apply -f -
kubectl expose deployment nginx -n revision-lab --port=80 --target-port=80 \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout status deployment/nginx -n revision-lab --timeout=120s

kubectl get deployment,replicaset,pod,service -n revision-lab -o wide
kubectl get pods -n revision-lab --show-labels
kubectl get endpointslices -n revision-lab -o yaml

kubectl run curl-test -n default --image=curlimages/curl \
  --rm -i --restart=Never -- curl -fsS --max-time 10 http://nginx.revision-lab

# Substitute one actual nginx Pod from revision-lab
kubectl delete pod POD_NAME -n revision-lab
kubectl get pods -n revision-lab -w
# Ctrl+C when you have observed the replacement

kubectl set image deployment/nginx -n revision-lab nginx=nginx:latest
kubectl rollout status deployment/nginx -n revision-lab --timeout=120s
kubectl get replicasets -n revision-lab
kubectl rollout history deployment/nginx -n revision-lab
kubectl rollout undo deployment/nginx -n revision-lab
kubectl rollout status deployment/nginx -n revision-lab --timeout=120s
kubectl get deployment nginx -n revision-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Optional teardown: `kubectl delete namespace revision-lab` deletes the entire exercise namespace and its resources. To retain everything for later, just stop Minikube.

</details>

## Connect this to the PostgreSQL example

The same concepts apply to your [existing manifests](postgres/README.md):

```text
Deployment → one PostgreSQL Pod
Secret → password supplied through an environment-variable reference
PVC → requested persistent storage → mounted in the container
Service → app=postgres Pods → PostgreSQL port 5432
```

**Secret:** Kubernetes object for sensitive configuration. **PVC:** a namespaced request for persistent storage; backing storage must be available/provisioned. Practise these in [configuration and health](12-configuration-and-health.md) and [storage](13-storage-and-state.md).

The [Service](postgres/service.yaml) is a ClusterIP on port `5432` targeting the named container port `postgres` (5432). The [Deployment](postgres/deployment.yaml) uses one replica, Recreate updates, startup/readiness probes, resources, and the [PVC](postgres/storage.yaml), all in `db-lab`. The [walkthrough](postgres/README.md) proves SQL access, persistence, and a separate logical restore.

A PVC can preserve data across Pod replacement; it does not by itself provide backup, database HA, or survival after destroying local backing storage. PostgreSQL needs a database client; an HTTP URL does not test its protocol.

## A small revision loop

**Today:** redraw the picture and answer the questions. **In a few days:** repeat missed questions. **Later:** run the exercise from the prompts before opening its solution. Keep additions small: one concept, one command, one question at a time.

## Independent capstone

Use `revision-lab` only if it is yours to change. You may reuse the shorter revision exercise's namespace after inspecting it. Write your own files in a new working directory; do not copy the guided manifests or open the worked sequence during this attempt. `kubectl explain` and official API documentation are allowed references. Check the current context first.

### Requirements

1. Declare a namespace, a three-replica HTTP Deployment, and a ClusterIP Service. Choose Service port **9090** with backend port **80** so you must reason about port mapping. Label ownership and Service selection correctly.
2. Serve a recognizable page from a ConfigMap directory mount. Inject another ConfigMap value as an environment variable. Declare requests/limits, startup/readiness checks, and a justified liveness decision. Avoid unnecessary automatic API token mounting.
3. From a Pod in another namespace, resolve the Service and request the page using its qualified name and Service port. Explain which parts of this test check DNS, forwarding, and application behavior.
4. Replace one Pod and prove a new UID appears while the Deployment and Service remain. Scale down/up and explain why scaling is different from a template revision.
5. Introduce a failed template revision, preserve evidence, and recover. Verify the page after recovery; do not equate a completed rollout with correct content.
6. Change the ConfigMap and demonstrate file refresh versus environment snapshot behavior. Document any needed application reload/restart.
7. In a separate database namespace, complete the PostgreSQL write/replacement/restore exercise. Explain the storage lifetime and why database rollback is separate from Deployment rollback.
8. Create a ServiceAccount with only ConfigMap read access in the challenge namespace. Prove the intended read works and Secret/write/cross-namespace actions are denied. Explain impersonation prerequisites.

### Diagnose without the worked commands

Inject one fault at a time and restore health before the next: a selector matching no Pods, a wrong target port, a missing required ConfigMap key, an unschedulable CPU request, and a failed readiness check. A partner may choose the fault without telling you; if working alone, record the mutation and return later without reading it.

For each fault, write the symptom and caller location, two plausible causes, a check distinguishing them, relevant output, the repair, and a successful verification. Never destroy a database claim or cluster to fix an ordinary workload fault.

### Completion evidence

| Area | Required evidence |
| --- | --- |
| Explanation | Draw API/control loops and a client request path without notes |
| Reproduction | Saved manifests plus tool/runtime versions and resolved image identities |
| Workloads | Before/after UID, replica intent, failed revision evidence, recovery |
| Networking | Qualified DNS result, Service/backend ports, actual page response |
| Configuration/health | Environment/file comparison; readiness consequences and restart distinction |
| Storage | Row before/after replacement and in a separately restored database |
| Access | An allowed read and denied operations with the tested identity named |
| Diagnosis | A justified hypothesis, discriminating check, repair, verification for each fault |

Pass only when every row is supported and you can repeat a missed part without its solution. If you needed a copied fix, that part is practice rather than an independent pass. Do not claim zero downtime, high availability, or disaster recovery from these local results.

### After this foundation

Choose a next topic based on the problem you need to solve: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) and [Gateway API](https://gateway-api.sigs.k8s.io/) for HTTP routing/TLS; [Jobs/CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) for completion/scheduling; [scheduling constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) for placement; [autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/) for demand; [observability](https://kubernetes.io/docs/concepts/cluster-administration/observability/) for monitoring/logging/tracing; and multi-node operation for real failure domains. The separate [CI/CD backlog](11-ci-cd-backlog.md) comes after you can deploy and recover manually.
