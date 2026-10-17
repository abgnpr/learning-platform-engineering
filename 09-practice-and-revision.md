# 9. Command sheet and practice

[Index](README.md) · Previous: [Recall questions](08-interview-preparation.md) · Next: [YAML manual](10-kubernetes-yaml-manual.md)

## Read kubectl commands once, then recognize the pattern

```text
kubectl VERB RESOURCE NAME -n NAMESPACE OPTIONS
        │       │      │        │
        action  kind   object   scope
```

For example, `kubectl describe pod nginx-abc -n demo` means **describe → this Pod → in demo**.

| Option | Meaning |
| --- | --- |
| `-n demo` | Target namespace demo |
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

Examples target namespace `demo` and app/container `nginx`. Replace uppercase placeholders. Create/expose commands require their names to be unused; inspect first.

### Orient and inspect

```bash
minikube start
minikube status
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,svc -A
kubectl get deployment,replicaset,pod,service -n demo
kubectl get pods -n demo -o wide --show-labels
kubectl describe pod POD_NAME -n demo  # Object state and events
kubectl logs POD_NAME -n demo          # Current application output
kubectl logs POD_NAME -n demo --previous # Previous container instance
```

### Declare and change intent

```bash
kubectl create namespace demo
kubectl create deployment nginx -n demo --image=nginx:alpine
kubectl apply -f FILE.yaml
kubectl scale deployment nginx -n demo --replicas=3
kubectl set image deployment/nginx -n demo nginx=nginx:latest
kubectl rollout status deployment/nginx -n demo --timeout=120s
kubectl rollout history deployment/nginx -n demo
kubectl rollout undo deployment/nginx -n demo
```

### Connect and debug

```bash
kubectl expose deployment nginx -n demo --port=80 --target-port=80
kubectl describe svc nginx -n demo
kubectl get endpointslices -n demo -l kubernetes.io/service-name=nginx -o yaml
kubectl port-forward -n demo svc/nginx 8080:80 # Local 8080 → Service 80 → targetPort
```

For a NodePort Service, create it with `--type=NodePort`; use `minikube service nginx -n demo --url` to find the local access URL. For DNS/application test Pods, see [the three test commands](05-dns-and-namespaces.md#test-from-inside-the-cluster).

### Stop versus delete

```bash
minikube stop                         # Retain cluster state
kubectl delete pod POD_NAME -n demo   # Managed Pod can be replaced automatically
```

`kubectl delete namespace NAME` deletes resources in that namespace. `minikube delete -p PROFILE` destroys the local profile. Use these only for intentional teardown.

## Practise predicting, not just typing

Use `revision-lab` for this exercise so existing `demo` resources can stay separate. Confirm `kubectl config current-context` is your local Minikube context. Exercise commands change the cluster and are not run by reading these notes.

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

**Secret:** Kubernetes object for sensitive configuration. **PVC:** a namespaced request for persistent storage; backing storage must be available/provisioned. These connect to configuration and storage topics for later study.

The [Service](postgres/service.yaml) maps NodePort `32432` to Service/target port `5432`. The [Deployment](postgres/deployment.yaml) uses requests/limits and mounts the [PVC](postgres/storage.yaml). Identify these relationships in the YAML rather than memorizing filenames.

A PVC can preserve data across Pod replacement; it does not by itself provide backup, database HA, or survival after destroying local backing storage. PostgreSQL needs a database client; an HTTP URL does not test its protocol.

## A small revision loop

**Today:** redraw the picture and answer the questions. **In a few days:** repeat missed questions. **Later:** run the exercise from the prompts before opening its solution. Keep additions small: one concept, one command, one question at a time.
