# 8. Recall first, then explain

[Index](README.md) · Previous: [Troubleshooting](07-troubleshooting.md) · Next: [Command sheet and practice](09-practice-and-revision.md)

**Hide the answers. Explain each group aloud in plain language.** For interviews, add one command that could verify your explanation. Mark only the questions you miss and revisit those first.

First-time prerequisite: complete the guided learning route through troubleshooting. Recall checks test explanation; the [independent capstone](09-practice-and-revision.md#independent-capstone) separately tests execution and diagnosis.

## 1. Tools and architecture

1. What does Minikube do? What does kubectl do?
2. How can Podman and CRI-O/containerd both be part of the same setup?
3. Who accepts, stores, schedules, and runs a workload?
4. Why is API acceptance different from application readiness?

<details>
<summary>Answer cues</summary>

1. Local cluster management / Kubernetes API client.
2. Podman supplies the node environment; the configured CRI-O/containerd runtime inside it runs workload containers. Verify the runtime rather than inferring it from the driver.
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
5. From `default`, how do you reach Service `nginx` in `foundation`?
6. Do namespaces automatically isolate network traffic?

<details>
<summary>Answer cues</summary>

1. One node / one Pod / stable virtual Service destination.
2. Selector matching Pod labels in the same namespace. Selection does not imply ownership.
3. Service entry / backend destination / optional node-facing entry.
4. Backend address, port, and condition data consumed by networking.
5. `nginx.foundation`, with the Service's port.
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
kubectl logs POD_NAME -n foundation --previous
kubectl describe pod POD_NAME -n foundation
kubectl describe svc nginx -n foundation
kubectl get pods -n foundation --show-labels
kubectl get endpointslices -n foundation -o yaml
type -a minikube
```

Pick the command that answers the unresolved question. Do not run every command mechanically.

</details>

## One-minute explanation challenge

**“Explain what happens from creating nginx to another Pod reaching it by name.”**

Your answer should connect API → controllers → scheduling → kubelet/runtime → labels → Service backends → DNS → port mapping. If one connection feels vague, revise that topic rather than memorizing a longer speech.

## 6. Configuration, health, storage, and access

1. A ConfigMap changes: predict the effect on an environment variable, a directory-mounted file, and a `subPath` file mount.
2. Readiness fails while liveness succeeds: what happens to traffic eligibility, Pod UID, and restarts?
3. Liveness fails repeatedly: what should restart, and what identity normally remains?
4. Distinguish container writable storage, `emptyDir`, PVC-backed storage, and a backup.
5. Does `ReadWriteOnce` guarantee that only one Pod can write? Does a StatefulSet implement database replication?
6. Why might a changed password Secret not change an initialized PostgreSQL role password?
7. Distinguish ServiceAccount, Role, RoleBinding, and container `runAsUser`.
8. Does an accepted NetworkPolicy establish that traffic is blocked?

<details>
<summary>Answer cues</summary>

1. Existing environment stays unchanged; projected directory updates eventually and needs app reread/reload; subPath does not receive normal updates.
2. Pod becomes unready for ordinary Service traffic; UID remains and readiness alone does not restart it.
3. The container restarts under its policy; normally the same Pod UID.
4. Container-local changes are disposable; emptyDir lasts for the Pod; PVC data depends on claim/backend lifetime; a tested backup supports separate recovery.
5. No: one node, potentially several Pods. No: StatefulSet manages identities/storage relationships; the database still needs replication logic.
6. The image's environment initializes an empty data directory; existing database credentials need a coordinated database-level change.
7. Workload API identity; allowed API actions; grant to a subject; Linux process UID. These control different boundaries.
8. No: a supporting network implementation must enforce it, and a connection experiment must verify behavior.

</details>

## Rate evidence, not familiarity

For each topic record **0: cannot explain**, **1: explain with prompts**, **2: explain and demonstrate unaided**, or **3: predict, diagnose an unfamiliar failure, and recover**. Mark a foundation complete only when every core topic reaches 2 and workload/network/configuration diagnosis reaches 3. A high average must not hide a missing storage or access foundation.

Retest with one changed condition: a new namespace, a different Service port, a missing configuration key, or an image pull failure. Explain what observation would disprove your first hypothesis.
