# 7. Troubleshooting: find the failing layer

[Index](README.md) · Previous in learning route: [Security and access](14-security-and-access.md) · Next: [Recall questions](08-interview-preparation.md)

**Goal:** use observations to distinguish hypotheses and recover with the smallest relevant change. First-time prerequisites: complete [configuration/health](12-configuration-and-health.md), [storage](13-storage-and-state.md), and [access](14-security-and-access.md). The saved guided Service listens on **8080**.

## Keep a short incident record

Write **symptom + caller location → two possible causes → discriminating check → evidence → fix → verification**. For example, a failed HTTP request could be DNS failure or a bad target port. `nslookup` distinguishes name resolution from later connection stages. A timeout alone does not name the failing component.

Capture events/logs before restarting. After a change, rerun both the failed check and a relevant application request. A successful object update does not establish recovery. Put persistent intended changes into your saved manifests so the next apply does not undo the repair.

> **Remember:** Target → workload → selector/backends → name/port → application. Choose the next check from the actual error.

## Start with the two easiest mistakes

```bash
kubectl config current-context   # Right cluster?
kubectl get pods,svc -A           # Right namespace and object name?
```

A missing object in one namespace does not mean it is absent everywhere. Do these checks before changing configuration.

## Match the symptom to a useful command

| Symptom | First useful check | What to look for |
| --- | --- | --- |
| Updated CLI still old | `type -a minikube` | Another executable, alias, or function selected |
| Node container missing | `sudo podman ps -a` | Rootful vs rootless store; stopped container |
| Pod Pending / creating | `kubectl describe pod POD_NAME -n foundation` | Scheduling, image, network, volume events |
| Container keeps restarting | `kubectl logs POD_NAME -n foundation --previous` | Last container's application error; also inspect exit reason |
| Pod runs but app unavailable | `kubectl describe pod POD_NAME -n foundation` | Ready condition and any probe failure |
| Service has no usable backend | EndpointSlice query below | Selector match, addresses, endpoint readiness |
| Could not resolve host | DNS test in [DNS notes](05-dns-and-namespaces.md#test-from-inside-the-cluster) | Exact Service name and caller namespace |
| Rollout stalls | `kubectl rollout status deployment/nginx -n foundation --timeout=120s` | New Pods not progressing; inspect those Pods |
| Resources look inconsistent | Compare configuration, node status, and cgroups | Different layers answering different questions |

`logs` shows application output. `describe` shows object state and events. Use both rather than expecting one to explain every failure.

## A Service is unreachable: walk the chain

```text
Am I testing from the laptop or from a Pod?
                    ↓
Does the Service exist in the intended namespace?
                    ↓
Does its selector match ready Pods?
                    ↓
Do EndpointSlices have eligible addresses and the correct port?
                    ↓
Does the exact DNS name resolve from the caller?
                    ↓
Can traffic reach the application listener?
```

```bash
# Object and application state
kubectl get pods -n foundation -o wide
kubectl describe pod POD_NAME -n foundation
kubectl logs POD_NAME -n foundation
kubectl get events -n foundation --sort-by=.metadata.creationTimestamp

# Connection between the Service and Pods
kubectl get svc nginx -n foundation -o yaml
kubectl get pods -n foundation -l app=nginx --show-labels
kubectl get endpointslices -n foundation -l kubernetes.io/service-name=nginx -o yaml
```

Compare **Service selector → Pod labels**, then **Service targetPort → actual listening port**. An address being present in an EndpointSlice does not by itself make it ready.

## Use failures to narrow the search

| What works? | What fails? | Likely area to investigate next |
| --- | --- | --- |
| Qualified name `nginx.foundation` | Short name `nginx` from another namespace | Namespace/search-path assumption |
| Service-IP access | Service-name access | DNS/resolver configuration |
| Application through port-forward | Normal Service access | Service selection/forwarding, routing, policy; port-forward bypasses the normal VIP path |
| Cluster-internal access | Laptop access | Exposure type, driver networking, host routing/firewall |
| Old Pods | New rollout Pods | New image/configuration, readiness, scheduling or capacity |

These comparisons narrow the investigation; they are not automatic root-cause proofs.

## Five traps to avoid

1. **Running ≠ ready.** Inspect readiness and make an application request.
2. **Deleting a managed Pod ≠ scaling down.** Change desired replicas.
3. **get all ≠ every object.** Query the relevant resource types explicitly.
4. **Rollout complete ≠ application correctness or zero downtime.** Verify what the user experiences.
5. **A successful response does not explain a later warning.** Read exact errors, retry targeted checks, and preserve evidence.

## Guided failure drills

Use your local cluster and `foundation` only. Begin with a healthy nginx Deployment and restored Service. If you skipped earlier labs, apply namespace, base Deployment, and Service in that order using the [lab guide](labs/kubernetes/README.md). After chapter 12, use its configuration stage instead of the base Deployment. Reuse the temporary HTTP client pattern from chapter 4 to test `http://nginx:8080`.

### Drill A: DNS resolves, but the backend port is wrong

**Predict:** will names and endpoint addresses disappear if only `targetPort` changes?

```bash
kubectl patch service nginx -n foundation --type=json \
  -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":81}]'
```

Make the HTTP request and record the error. Before repairing, inspect the Service YAML, EndpointSlice port, and Pod container port/listener expectation. The Pods can remain healthy; endpoint addresses can exist while their destination port is wrong. Port-forward to the **Deployment's** port 80 can help compare application versus Service configuration, but is not proof of the Service VIP path.

**Recover:** reapply `labs/kubernetes/service.yaml`, then repeat the in-cluster request. Expect Service port 8080 and endpoint port 80 again.

### Drill B: a process repeatedly exits

```bash
kubectl apply -f labs/kubernetes/crash-demo.yaml
kubectl get pod crash-demo -n foundation -w
```

Stop the watch after restarts begin. Collect current/previous logs and `describe` before deleting it:

```bash
kubectl logs crash-demo -n foundation
kubectl logs crash-demo -n foundation --previous
kubectl describe pod crash-demo -n foundation
```

Expect a deliberate error line, termination exit code 1, restarts, and eventually a backoff summary. `--previous` requires a previous container instance; if it is not available yet, wait for one restart. Distinguish an application exit from a liveness-triggered restart using events and termination state. This bare Pod has no controller to replace it after deletion.

**Recover/cleanup:** the fixture intentionally always fails; delete it with `kubectl delete pod crash-demo -n foundation`. For a real workload, correct the process/configuration rather than relying on endless restarts.

### Drill C: no capacity, then no storage

Repeat the [resource scheduling failure](06-resources-and-cgroups.md#lab-low-usage-does-not-imply-scheduling-capacity) from prompts without its solution. Contrast its events with the storage checks in the [PostgreSQL lab](postgres/README.md). Explain why both can yield Pending while requiring different fixes; do not remove the database PVC as a troubleshooting shortcut.

**Checkpoint:** for each drill, show one observation that rules out an alternative explanation. Give the recovery command and the application-level verification. A list of commands without that reasoning is not a pass.

## Practise choosing the next check

1. There are three ready Pods but zero Service backends. What do you compare?
2. A Pod in `default` cannot resolve `nginx`; the Service exists in `foundation`. What name do you try?
3. A tool update completed, but its version is unchanged. What do you inspect before reinstalling?

<details>
<summary>Answers</summary>

1. Service namespace and selector against Pod namespace/labels; then EndpointSlice conditions and ports.
2. `nginx.foundation` using the Service's port.
3. Command resolution/PATH, then absolute-path versions and package ownership.

</details>

**Useful interview habit:** say the symptom, show the evidence, name the hypothesis, and choose a check that could disprove it. Restarting or deleting everything first loses the chance to understand the failure.
