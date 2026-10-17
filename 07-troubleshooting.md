# 7. Troubleshooting: find the failing layer

[Index](README.md) · Previous: [Resources](06-resources-and-cgroups.md) · Next: [Recall questions](08-interview-preparation.md)

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
| Pod Pending / creating | `kubectl describe pod POD_NAME -n demo` | Scheduling, image, network, volume events |
| Container keeps restarting | `kubectl logs POD_NAME -n demo --previous` | Last container's application error; also inspect exit reason |
| Pod runs but app unavailable | `kubectl describe pod POD_NAME -n demo` | Ready condition and any probe failure |
| Service has no usable backend | EndpointSlice query below | Selector match, addresses, endpoint readiness |
| Could not resolve host | DNS test in [DNS notes](05-dns-and-namespaces.md#test-from-inside-the-cluster) | Exact Service name and caller namespace |
| Rollout stalls | `kubectl rollout status deployment/nginx -n demo --timeout=120s` | New Pods not progressing; inspect those Pods |
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
kubectl get pods -n demo -o wide
kubectl describe pod POD_NAME -n demo
kubectl logs POD_NAME -n demo
kubectl get events -n demo --sort-by=.metadata.creationTimestamp

# Connection between the Service and Pods
kubectl get svc nginx -n demo -o yaml
kubectl get pods -n demo -l app=nginx --show-labels
kubectl get endpointslices -n demo -l kubernetes.io/service-name=nginx -o yaml
```

Compare **Service selector → Pod labels**, then **Service targetPort → actual listening port**. An address being present in an EndpointSlice does not by itself make it ready.

## Use failures to narrow the search

| What works? | What fails? | Likely area to investigate next |
| --- | --- | --- |
| Qualified name `nginx.demo` | Short name `nginx` from another namespace | Namespace/search-path assumption |
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

## Practise choosing the next check

1. There are three ready Pods but zero Service backends. What do you compare?
2. A Pod in `default` cannot resolve `nginx`; the Service exists in `demo`. What name do you try?
3. A tool update completed, but its version is unchanged. What do you inspect before reinstalling?

<details>
<summary>Answers</summary>

1. Service namespace and selector against Pod namespace/labels; then EndpointSlice conditions and ports.
2. `nginx.demo` using the Service's port.
3. Command resolution/PATH, then absolute-path versions and package ownership.

</details>

**Useful interview habit:** say the symptom, show the evidence, name the hypothesis, and choose a check that could disprove it. Restarting or deleting everything first loses the chance to understand the failure.
