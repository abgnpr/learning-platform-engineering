# 4. Services and networking

[Index](README.md) · Previous: [Workloads](03-workloads-and-rollouts.md) · Next: [DNS](05-dns-and-namespaces.md)

## Remember these four things

- **Node IP = where a node is. Pod IP = where one Pod is.**
- **Service = a stable destination for changing Pods.**
- **Labels + selector = how the Service finds its Pods.**
- **port → targetPort = Service port → application port.**

## Why a Service exists

**Goal:** follow a request from a stable destination to a changing backend. Prerequisite: the healthy `foundation/nginx` Deployment from chapter 3. First read the address/port explanation; the lab intentionally uses different Service and backend ports.

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
| `port` | `8080` | Port a client uses on the Service |
| `targetPort` | `80` | Port the application listens on in a backend Pod |
| `nodePort` | `31757` | Extra entry port through eligible node addresses |

The examples use **8080 → 80**: clients call the Service on 8080 while nginx listens on 80. Matching numbers such as 80 → 80 are also valid; the two fields still describe different roles. Declaring a port does not make the application listen.

```text
Inside cluster:  nginx:8080 → Service forwarding → PodIP:80
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

Assume Deployment `nginx` exists in `foundation`. Choose one expose command when creating its Service.

```bash
# Internal Service: Service port 8080 forwards to nginx port 80
kubectl expose deployment nginx -n foundation --port=8080 --target-port=80

# Alternative: also expose it through a NodePort
kubectl expose deployment nginx -n foundation --type=NodePort --port=8080 --target-port=80

kubectl get svc -n foundation             # Type, ClusterIP, exposed ports
kubectl describe svc nginx -n foundation  # Selector, target port, backends
minikube service nginx -n foundation --url # Discover an access URL for local exposure

# Temporary laptop debugging path: localhost:8080 → a selected Pod's port 80
kubectl port-forward -n foundation svc/nginx 8080:8080
```

Port forwarding runs until stopped. Its second port refers to the **Service port**, which is then mapped to the target port. It bypasses normal Service VIP load balancing.

## EndpointSlice = the backend list

**Service says who to match. EndpointSlices record the backend addresses, ports, and conditions.** Networking consumes that information to forward traffic.

```bash
kubectl get pods -n foundation -l app=nginx -o wide
kubectl get endpointslices -n foundation -l kubernetes.io/service-name=nginx -o yaml
```

Compare the addresses with Pod IPs and inspect endpoint readiness. EndpointSlices are data, not another packet hop. Prefer them over the deprecated legacy `Endpoints` API.

<details>
<summary>Details to revisit only when needed</summary>

With `externalTrafficPolicy: Cluster`, the receiving node can forward to a Pod on another node. `Local` restricts external traffic to local eligible backends. Existing connections and affinity affect backend selection; repeated requests need not visibly alternate evenly. A Service with `publishNotReadyAddresses` changes normal readiness handling.

</details>

## Lab: follow a request, then remove its backends

The saved guided-lab Service uses **8080 → named port `http` → 80**. Use the file below to create the Service; the earlier expose commands explain equivalent creation choices and need not be run first.

**Predict:** will a Service named nginx select Pods because their names start with nginx, or because their labels match?

```bash
kubectl apply -f labs/kubernetes/service.yaml
kubectl get svc nginx -n foundation
kubectl get endpointslices -n foundation -l kubernetes.io/service-name=nginx -o yaml
kubectl run network-client -n foundation --image=busybox:1.37.0 \
  --rm -i --restart=Never -- wget -T 5 -qO- http://nginx:8080
```

Expect a ClusterIP, EndpointSlices with Pod addresses/port 80 and eligible conditions, and an HTML response. Initially, without an application readiness probe, container startup supplies only a weaker readiness signal; chapter 12 improves it.

In a second terminal you can also run:

```bash
kubectl port-forward -n foundation svc/nginx 8080:8080
```

From the host, `curl --fail --max-time 5 http://127.0.0.1:8080` should work while forwarding runs. This selects a Pod through the API forwarding path; it does not prove that the Service VIP path works. Stop forwarding with Ctrl+C before the next experiment. If 8080 is occupied, use local `18080:8080` and update the host URL.

**Break selection without changing any Pods:**

```bash
kubectl patch service nginx -n foundation --type=merge \
  -p '{"spec":{"selector":{"app":"no-such-app"}}}'
kubectl get pods -n foundation -l app=nginx
kubectl get endpointslices -n foundation -l kubernetes.io/service-name=nginx -o yaml
kubectl run network-client -n foundation --image=busybox:1.37.0 \
  --rm -i --restart=Never -- wget -T 5 -qO- http://nginx:8080
```

Expect healthy Pods but no eligible backend addresses after reconciliation. DNS can still resolve the Service; the request fails. The exact refusal/timeout depends on the networking implementation. Do not infer that the application crashed merely from failed Service access.

**Recover:** reapply `labs/kubernetes/service.yaml`, inspect EndpointSlices until backends return, and repeat the request. Interrupted test Pods can leave a name behind; inspect and delete only that test Pod before rerunning.

**Checkpoint:** explain every port and arrow in `client → nginx:8080 → backend Pod:80`. Prove that application health and Service selection are separate questions. Leave the restored Service for DNS practice.

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
