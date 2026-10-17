# 8. Recall first, then explain

[Index](README.md) · Previous: [Troubleshooting](07-troubleshooting.md) · Next: [Command sheet and practice](09-practice-and-revision.md)

**Hide the answers. Explain each group aloud in plain language.** For interviews, add one command that could verify your explanation. Mark only the questions you miss and revisit those first.

## 1. Tools and architecture

1. What does Minikube do? What does kubectl do?
2. How can Podman and containerd both be part of the same setup?
3. Who accepts, stores, schedules, and runs a workload?
4. Why is API acceptance different from application readiness?

<details>
<summary>Answer cues</summary>

1. Local cluster management / Kubernetes API client.
2. Podman supplies the node environment; containerd inside it runs workload containers.
3. API server accepts; etcd stores; controllers reconcile; scheduler assigns a node; kubelet works through the runtime to run containers.
4. Scheduling and image/network/volume/application startup happen afterward. Check rollout status, Pod conditions, events, and an application request.

</details>

## 2. Workload lifecycle

1. Explain Deployment → ReplicaSet → Pod.
2. What happens when you delete one Pod from a three-replica Deployment?
3. Which change creates a rollout: scaling or changing the image?
4. Why can four Pods appear when replicas is three?
5. What does rollback restore, and what does it leave outside its scope?

<details>
<summary>Answer cues</summary>

1. Rollout intent → maintain count → run containers.
2. The ReplicaSet creates a replacement; it is a new Pod, potentially with a new IP.
3. Image change alters the Pod template. Scaling alone does not create a template revision.
4. Surge capacity: default 25% of three rounds up to one extra Pod.
5. An earlier Pod template. Database contents and external side effects are not restored by Deployment undo.

</details>

## 3. Networking and names

1. Distinguish Node IP, Pod IP, and Service ClusterIP.
2. What connects a Service to Pods? Does the Service own them?
3. Explain port, targetPort, and nodePort.
4. What is an EndpointSlice?
5. From `default`, how do you reach Service `nginx` in `demo`?
6. Do namespaces automatically isolate network traffic?

<details>
<summary>Answer cues</summary>

1. One node / one Pod / stable virtual Service destination.
2. Selector matching Pod labels in the same namespace. Selection does not imply ownership.
3. Service entry / backend destination / optional node-facing entry.
4. Backend address, port, and condition data consumed by networking.
5. `nginx.demo`, with the Service's port.
6. No. Additional policy and its enforcement are needed.

</details>

## 4. Resources and evidence

1. Decode `250m`, `500m`, and `512Mi`.
2. Request versus limit: what happens under CPU or memory pressure?
3. Is node capacity a live usage measurement?
4. Does `cpu.max = max 100000` prove the process can use every CPU?
5. Does successful rollout status prove zero downtime?

<details>
<summary>Answer cues</summary>

1. 0.25 CPU, 0.5 CPU, 512 × 1024² bytes.
2. Requests guide scheduling/allocation. Limits constrain usage; CPU commonly throttles, memory excess can trigger OOM kills.
3. No. Capacity/allocatable, requested/limited totals, and actual usage are different views.
4. No bandwidth quota at that cgroup; cpusets, affinity, ancestors, and contention may constrain it.
5. No. It establishes rollout conditions; application continuity requires separate observation.

</details>

## 5. Choose a command, not a guess

| Situation | Name your next check before opening the answer |
| --- | --- |
| nginx exists somewhere but `get pods` shows nothing | Cluster/namespace? |
| Pod is restarting | Previous logs and termination reason? |
| Three ready Pods, no Service backends | Namespace/selector/labels? |
| DNS works but connection fails | Backends/ports/listener/routing? |
| New binary installed, old version printed | Executable resolution? |

<details>
<summary>Command cues</summary>

```bash
kubectl config current-context
kubectl get pods,svc -A
kubectl logs POD_NAME -n demo --previous
kubectl describe pod POD_NAME -n demo
kubectl describe svc nginx -n demo
kubectl get pods -n demo --show-labels
kubectl get endpointslices -n demo -o yaml
type -a minikube
```

Pick the command that answers the unresolved question. Do not run every command mechanically.

</details>

## One-minute explanation challenge

**“Explain what happens from creating nginx to another Pod reaching it by name.”**

Your answer should connect API → controllers → scheduling → kubelet/runtime → labels → Service backends → DNS → port mapping. If one connection feels vague, revise that topic rather than memorizing a longer speech.
