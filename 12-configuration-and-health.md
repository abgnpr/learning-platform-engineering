# 12. Configuration and health: running is only the beginning

[Index](README.md) · Learning route: [Resources](06-resources-and-cgroups.md) → this lesson → [Storage](13-storage-and-state.md)

**Goal:** supply configuration separately from an image and prove the different consequences of failed health checks. Prerequisites: the `foundation/nginx` Deployment and Service from chapters 3–6. Run from the repository root. Service port is **8080**, backend nginx port is **80**.

## Why separate configuration from an image?

The same application image can run in different environments with different settings. A **ConfigMap** stores ordinary configuration; a **Secret** holds sensitive values intended for controlled access. Neither automatically teaches an application how to use a variable or reload a file.

```text
ConfigMap key ── env.valueFrom ──→ environment at container start
ConfigMap data ─ volume/mount ──→ files visible to the application
Secret key ───── secretKeyRef ──→ sensitive environment value
```

| Delivery method | After changing the source object |
| --- | --- |
| Environment variable | Existing process environment is unchanged; replace/restart the container to receive the new value |
| ConfigMap/Secret directory volume | Projected files update eventually; application must reread/reload them |
| A `subPath` file mount | Does not receive normal projected-volume updates |

A ConfigMap edit by itself does not change a Deployment's Pod template and does not automatically create a rollout. Environment and file behavior differ. A missing required object/key can prevent startup; read the Pod events.

Secret `data` uses base64 encoding, which is not encryption. `stringData` accepts readable input. Kubernetes Secrets need appropriate RBAC and encryption-at-rest configuration; storing a value in a Secret does not establish those controls. Use fake data for these labs and keep real credentials out of Git and logs. See [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) and [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

## Lab A: environment snapshot versus projected file

Read [configuration.yaml](labs/kubernetes/configuration.yaml). It replaces the base Deployment definition, adds probes/resources, injects `MESSAGE`, and mounts the ConfigMap as nginx's web directory. nginx serves the HTML file; it does not automatically display the environment variable.

**Predict:** what will change immediately after patching the ConfigMap? Will Pod UIDs change?

```bash
kubectl apply -f labs/kubernetes/configuration.yaml
kubectl rollout status deployment/nginx -n foundation --timeout=180s
kubectl get pods -n foundation -l app=nginx
```

Choose one actual Pod name for `POD_NAME`. Record its UID and environment:

```bash
kubectl get pod POD_NAME -n foundation -o jsonpath='{.metadata.uid}{"\n"}'
kubectl exec -n foundation POD_NAME -- printenv MESSAGE
kubectl exec -n foundation POD_NAME -- cat /usr/share/nginx/html/index.html
kubectl patch configmap nginx-content -n foundation --type=merge \
  -p '{"data":{"MESSAGE":"version two","index.html":"<h1>Foundation: version two</h1>\n"}}'
kubectl exec -n foundation POD_NAME -- printenv MESSAGE
kubectl exec -n foundation POD_NAME -- cat /usr/share/nginx/html/index.html
```

Expect the existing environment to remain `version one`. The file initially may also show the old contents, then update after kubelet projection; repeat the file check rather than assuming an exact delay. Pod UID remains unchanged. Every replica has its own projection timing, so requests may briefly see different content.

Verify the application path using a temporary client:

```bash
kubectl run config-client -n foundation --image=busybox:1.37.0 \
  --rm -i --restart=Never -- wget -T 5 -qO- http://nginx:8080
kubectl rollout restart deployment/nginx -n foundation
kubectl rollout status deployment/nginx -n foundation --timeout=180s
kubectl exec -n foundation deployment/nginx -- printenv MESSAGE
```

Expect the new HTML after projection, then `version two` in a new container's environment after rollout. `rollout restart` changes a Pod-template annotation and creates replacements; it does not reboot the node. Inspect newly listed Pod UIDs rather than using a deleted old Pod name.

**Recovery:** reapply `configuration.yaml`, then `kubectl rollout restart deployment/nginx -n foundation` and wait. Resetting the ConfigMap file alone would not reset an already-started environment. If the rollout fails, inspect required references, events, probes, and logs. Keep the ConfigMap-backed workload for later troubleshooting.

## Three different health questions

| Probe | Question | Consequence after its failure threshold |
| --- | --- | --- |
| Startup | Has initialization succeeded? | Restart this container; readiness/liveness checks wait until startup succeeds |
| Readiness | Should this Pod receive ordinary Service traffic now? | Mark it unready; ordinary Service backends stop treating it as eligible |
| Liveness | Is restarting this container an appropriate recovery? | Restart this container according to its restart policy |

Probe timing includes periods, thresholds, timeouts, and startup behavior; avoid interpreting a single failed check as an immediate restart. The kubelet performs these checks. A readiness failure does not itself restart a container. A liveness restart normally keeps the **same Pod UID**. A Deployment template change creates **new Pods**.

The nginx fixture uses HTTP `/` for simple checks because its job is serving one static page. A real app may need distinct startup/readiness/liveness endpoints. A dependency outage should not automatically trigger liveness restarts; restarting every instance may make an outage worse. Choose checks based on what the app can recover from, and keep them cheap. [Probe documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

## Lab B: readiness failure versus liveness restart

[health-demo.yaml](labs/kubernetes/health-demo.yaml) defines a separate nginx Pod and a Service selecting it. Its command creates `/tmp/ready` and `/tmp/live`; startup checks HTTP while readiness/liveness exec probes check the markers. Marker files are deliberate experiment controls, not a recommended production health implementation.

```bash
kubectl apply -f labs/kubernetes/health-demo.yaml
kubectl wait pod/health-demo -n foundation --for=condition=Ready --timeout=120s
kubectl get pod health-demo -n foundation \
  -o jsonpath='{.metadata.uid}{" restarts="}{.status.containerStatuses[0].restartCount}{"\n"}'
kubectl run health-client -n foundation --image=busybox:1.37.0 \
  --rm -i --restart=Never -- wget -T 5 -qO- http://health-demo:8080
kubectl exec -n foundation health-demo -- rm /tmp/ready
kubectl wait pod/health-demo -n foundation --for=condition=Ready=false --timeout=60s
kubectl get pod health-demo -n foundation
kubectl describe pod health-demo -n foundation
kubectl get endpointslices -n foundation -l kubernetes.io/service-name=health-demo -o yaml
kubectl run health-client -n foundation --image=busybox:1.37.0 \
  --rm -i --restart=Never -- wget -T 5 -qO- http://health-demo:8080
```

**Observe:** initially the request succeeds. After removing the readiness marker, the Pod is `Running` but `0/1` ready, its UID stays the same, and readiness causes no restart. After endpoint conditions reconcile, the Service has no eligible backend and the request fails even though nginx is still running. A stale endpoint condition may briefly precede the controller update; inspect and retry rather than assuming immediate propagation.

**Restore readiness, then fail liveness:**

```bash
kubectl exec -n foundation health-demo -- touch /tmp/ready
kubectl wait pod/health-demo -n foundation --for=condition=Ready --timeout=60s
kubectl run health-client -n foundation --image=busybox:1.37.0 \
  --rm -i --restart=Never -- wget -T 5 -qO- http://health-demo:8080
kubectl exec -n foundation health-demo -- rm /tmp/live
kubectl get pod health-demo -n foundation -w
```

Stop the watch with Ctrl+C after restart count increases. Recheck UID, restart count, and `describe` events. Expect the same UID and a higher restart count after failed liveness checks and container termination/startup. Termination grace can add time. On restart, the command recreates both markers and startup/readiness pass again. Inspect `.status.containerStatuses` if timing makes the brief unready state hard to catch.

**Recovery/cleanup:** after restoring readiness, wait for eligible endpoints if the first request still fails. If interrupted during failure, inspect and restore `/tmp/ready`, or delete/recreate only the health-demo Pod from its file. Final cleanup is `kubectl delete -f labs/kubernetes/health-demo.yaml`, removing both its Pod and Service. If an interrupted client remains, inspect and delete only `health-client` before reusing its name.

## Checkpoint without copying

1. Change a ConfigMap value and explain why a file and an environment variable behave differently.
2. Explain why the config-backed nginx rollout replaced Pods, while a liveness failure kept the health-demo UID.
3. A database dependency is briefly unavailable: justify readiness/liveness behavior without causing a restart storm.
4. Predict what happens if a required Secret key is misspelled. Identify the first command that could prove it.

**Pass evidence:** a before/after environment value, file content, a stable UID with increased restart count, and an explanation of why readiness alone did not restart anything. Continue with [storage and state](13-storage-and-state.md).
