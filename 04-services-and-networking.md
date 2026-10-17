# 4. Services and networking

[Index](README.md) · Previous: [Workloads](03-workloads-and-rollouts.md) · Next: [DNS](05-dns-and-namespaces.md)

## Remember these four things

- **Node IP = where a node is. Pod IP = where one Pod is.**
- **Service = a stable destination for changing Pods.**
- **Labels + selector = how the Service finds its Pods.**
- **port → targetPort = Service port → application port.**

## Why a Service exists

A replaced Pod can get a new IP. Applications should not have to rediscover individual Pods after every replacement. A Service keeps the client destination stable and selects eligible backends.

```text
                     Service: nginx
Client ────────────→ stable address ─────→ one eligible nginx Pod
                     selector: app=nginx   Pods carry app=nginx
```

The Service selects Pods in **its own namespace**. It does not own them; their ReplicaSet maintains them.

## Put addresses and ports in their boxes

| Thing | Example | Meaning |
| --- | --- | --- |
| Node IP | `192.168.49.2` | Address of one Kubernetes node |
| Pod IP | `10.244.0.3` | Address of one replaceable Pod |
| Service ClusterIP | `10.108.7.227` | Virtual address stable while this Service exists |
| `port` | `80` | Port a client uses on the Service |
| `targetPort` | `80` | Port the application listens on in a backend Pod |
| `nodePort` | `31757` | Extra entry port through eligible node addresses |

The examples below use **80 → 80**. The ports can differ: **8080 → 80** would mean clients call the Service on 8080 while nginx listens on 80. Declaring a port does not make the application listen.

```text
Inside cluster:  nginx:80 → Service forwarding → PodIP:80
Outside entry:   NodeIP:31757 → Service forwarding → PodIP:80
```

These are conceptual paths. A Service is an API object with programmed forwarding, not a separate server that receives every packet.

## Service types: learn two well

| Type | Remember |
| --- | --- |
| **ClusterIP** | Default; gives internal Service access |
| **NodePort** | Adds a node-facing port; still has a ClusterIP |
| LoadBalancer | Needs a supporting load-balancer implementation; local Minikube can use a tunnel |
| ExternalName | DNS alias; good to know for later |

A ClusterIP is not generally reachable directly from your laptop. NodePort reachability also depends on driver networking, routing, and firewall rules.

## Commands beside the idea

Assume Deployment `nginx` exists in `demo`. Choose one expose command when creating its Service.

```bash
# Internal Service: Service port 80 forwards to nginx port 80
kubectl expose deployment nginx -n demo --port=80 --target-port=80

# Alternative: also expose it through a NodePort
kubectl expose deployment nginx -n demo --type=NodePort --port=80 --target-port=80

kubectl get svc -n demo             # Type, ClusterIP, exposed ports
kubectl describe svc nginx -n demo  # Selector, target port, backends
minikube service nginx -n demo --url # Discover an access URL for local exposure

# Temporary laptop debugging path: localhost:8080 → a selected Pod's port 80
kubectl port-forward -n demo svc/nginx 8080:80
```

Port forwarding runs until stopped. Its second port refers to the **Service port**, which is then mapped to the target port. It bypasses normal Service VIP load balancing.

## EndpointSlice = the backend list

**Service says who to match. EndpointSlices record the backend addresses, ports, and conditions.** Networking consumes that information to forward traffic.

```bash
kubectl get pods -n demo -l app=nginx -o wide
kubectl get endpointslices -n demo -l kubernetes.io/service-name=nginx -o yaml
```

Compare the addresses with Pod IPs and inspect endpoint readiness. EndpointSlices are data, not another packet hop. Prefer them over the deprecated legacy `Endpoints` API.

<details>
<summary>Details to revisit only when needed</summary>

With `externalTrafficPolicy: Cluster`, the receiving node can forward to a Pod on another node. `Local` restricts external traffic to local eligible backends. Existing connections and affinity affect backend selection; repeated requests need not visibly alternate evenly. A Service with `publishNotReadyAddresses` changes normal readiness handling.

</details>

## Test your recall

1. Which address should survive replacing all nginx Pods?
2. A Service has `port: 8080`, `targetPort: 80`. Which port does the client call?
3. What connects the Service to Pods: matching names, ownership, or selectors?

<details>
<summary>Answers</summary>

1. The Service address, while that Service exists.
2. 8080 on the Service; traffic goes to port 80 on the Pod.
3. Its selector matching Pod labels in the same namespace.

</details>

References: [Services](https://kubernetes.io/docs/concepts/services-networking/service/), [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/).
