# 5. DNS and namespaces

[Index](README.md) · Previous: [Networking](04-services-and-networking.md) · Next: [Resources](06-resources-and-cgroups.md)

## Remember these four things

- **Namespace = scope for resource names.**
- **Short Service name = look in the caller's namespace.**
- **service.namespace = explicitly choose another namespace.**
- **Different namespace does not automatically mean blocked traffic.**

## Think of names as name + namespace

```text
Cluster
├── default/nginx    ← one Deployment and Service
└── demo/nginx       ← independent Deployment and Service
```

Resources of the same kind can share a name in different namespaces. Pods, Services, Deployments, Secrets, and PVCs are namespaced. Nodes and PersistentVolumes are cluster-scoped.

A namespace organizes names, permissions, and resource controls. It does not create a separate machine or automatically create network isolation.

## DNS: one rule, three spellings

For an ordinary Service named `nginx` in namespace `demo`:

| Name | Use it when |
| --- | --- |
| `nginx` | Caller is also in `demo` |
| `nginx.demo` | Caller may be in another namespace |
| `nginx.demo.svc.cluster.local` | You want the full name for a cluster using domain `cluster.local` |

```text
nginx . demo . svc . cluster.local
  │      │     │         └─ cluster DNS domain, configurable
  │      │     └─────────── Service zone
  │      └───────────────── namespace
  └──────────────────────── Service name
```

A normal Service name resolves to its **ClusterIP**. Creating only a Deployment does not create that Service DNS record.

**Example:** A Pod in `default` calls `nginx` → it looks for `default/nginx`. To reach `demo/nginx`, call `nginx.demo`.

## Commands to orient yourself

```bash
kubectl config current-context     # Which cluster/user context?
kubectl config get-contexts        # Also shows context default namespaces
kubectl get namespaces
kubectl get pods -n demo            # One namespace
kubectl get pods -A                 # All namespaces
kubectl get svc -A                  # Find where the Service actually exists
kubectl create namespace demo      # Create it if absent
```

A **context** groups a cluster, credentials/user, and optional default namespace. Without `-n`, kubectl uses the context's namespace or `default`. Always check both cluster and namespace when a resource seems missing.

## Test from inside the cluster

Assume `demo/nginx` exists on Service port 80. These create temporary test Pods in `default`:

```bash
# Application reachability using a namespace-qualified name
kubectl run curl-test -n default --image=curlimages/curl \
  --rm -i --restart=Never -- curl -fsS --max-time 10 http://nginx.demo

# DNS only; final dot asks for this exact full name
kubectl run dns-test -n default --image=busybox:1.36 \
  --rm -i --restart=Never -- nslookup nginx.demo.svc.cluster.local.

# Show the Pod's resolver and search domains
kubectl run resolver-test -n default --image=busybox:1.36 \
  --rm -i --restart=Never -- cat /etc/resolv.conf
```

`--rm`: remove the attached test Pod afterward. `-i`: attach input without a terminal. `--restart=Never`: use a standalone Pod that does not restart its completed container. After interruption, check for leftovers before reusing the name.

Use the Service's actual port in HTTP requests; if it is 8080, call `http://nginx.demo:8080`.

## Separate the failures

| Result | Investigate first |
| --- | --- |
| Could not resolve host / NXDOMAIN | Exact name, namespace, Service existence, DNS |
| Name resolves, connection fails | Backends, readiness, target port, application listener, routing/policy |
| HTTP response with an error | Application/proxy behavior and logs |

DNS search domains expand short names. Some expanded candidates may fail even when the intended name resolves. Read the exact queried names and exit status instead of treating all NXDOMAIN messages alike.

## Easy traps

- `kubectl get all` does **not** include every resource type. It can omit Secrets, ConfigMaps, and service accounts.
- Cluster DNS names generally resolve inside configured cluster networking; your laptop's normal DNS is separate.
- `kube-system` holds system resources. `kube-node-lease` holds node heartbeat leases. You normally create your app in your own namespace.

## Test your recall

1. From a Pod in `demo`, what does `http://nginx` try to reach?
2. How would that Pod call the nginx Service in `default`?
3. Does a Deployment named nginx guarantee nginx resolves?

<details>
<summary>Answers</summary>

1. The Service `demo/nginx`.
2. `http://nginx.default` using the Service's port.
3. No. The corresponding Service must exist; ordinary Service discovery names refer to Services.

</details>

References: [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), [Service DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).
