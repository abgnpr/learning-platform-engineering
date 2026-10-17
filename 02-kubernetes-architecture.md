# 2. Architecture: who does what?

[Index](README.md) · Previous: [Setup](01-linux-tooling-and-minikube.md) · Next: [Workloads](03-workloads-and-rollouts.md)

## Remember these four things

- **API accepts → etcd stores → controllers reconcile.**
- **Scheduler chooses where; kubelet makes it run there through the runtime.**
- **Cluster contains nodes; nodes run Pods; Pods contain containers.**
- **Running, Ready, and schedulable answer different questions.**

## Vocabulary in one picture

```text
Cluster
├── Control plane: cluster management
└── Nodes: execution locations
    └── Pods: replaceable workload units
        └── Containers: application processes
```

A Pod can contain multiple closely related containers sharing networking and declared volumes. Single-node Minikube combines control-plane and workload execution on one node.

## The components

| Component | Remember its job |
| --- | --- |
| API server | Front door for Kubernetes API operations |
| etcd | Persistent store of cluster API data |
| Controller manager | Controllers keep actual state moving toward desired state |
| Scheduler | Assigns an unscheduled Pod to a suitable node |
| kubelet | Ensures assigned Pods run on its node |
| containerd / runtime | Creates and runs the containers |
| CNI networking | Configures Pod networking |
| kube-proxy | Programs Service forwarding in this setup |
| CoreDNS | Resolves cluster names |
| Storage provisioner | Supplies volumes for supported storage requests |

The first four form the core control plane. Some networking implementations replace kube-proxy. See [architecture](https://kubernetes.io/docs/concepts/architecture/).

## Follow one create request

```text
kubectl → API server → stored desired Deployment
                              ↓
                    Deployment → ReplicaSet → Pods
                              ↓
                    scheduler assigns a node
                              ↓
                    kubelet → runtime → containers
```

These are cooperating loops, not a single synchronous call. **Accepted does not mean ready.** After the API accepts an object, scheduling, networking, images, and volumes may still need work.

Desired replicas = 3, actual Pods = 2 → the ReplicaSet controller creates another. A container restarting inside a Pod differs from replacing the whole Pod: replacement changes Pod identity and may change its IP.

## Read state with the right command

```bash
kubectl get nodes -o wide           # Nodes, health, addresses, kubelet versions
kubectl get pods -n kube-system     # System Pods
kubectl get pods -n demo -o wide    # App Pods, IPs, and node placement
kubectl describe pod POD_NAME -n demo # State, owner, conditions, events
kubectl logs POD_NAME -n demo       # Application output
kubectl describe node minikube      # Node conditions, taints, resources
```

| Pod field | Question answered |
| --- | --- |
| Node / IP | Where was it placed / what is this Pod's address? |
| Controlled By | Who maintains this Pod? |
| Container ID | Which runtime/container instance? |
| Image / Image ID | Requested reference / resolved image identity? |
| Ready / Restart Count | Ready now / how many recorded container restarts? |
| Events | What happened during scheduling and startup? |

Events such as `Scheduled → Pulled → Created → Started` help locate progress. Read messages: an image may already be cached. Persistent `ContainerCreating` can involve images, networking, or volumes.

## Three state distinctions

| Term | Meaning |
| --- | --- |
| Pod Running | Lifecycle phase; application requests can still fail |
| Pod Ready | Readiness conditions passed; application probes improve what this establishes |
| Node Ready / schedulable | Node health / whether scheduling is allowed, subject to constraints |

**Next step:** readiness probes test whether the app should receive traffic; liveness probes test whether it needs restarting. They serve different purposes.

A control-plane role label alone does not block workloads; taints and scheduling rules can. Three replicas on one laptop share one physical failure domain.

A Pod's service account is its API identity. Authorization such as RBAC determines what that identity may do.

## Test your recall

1. Who chooses the node, and who runs the assigned containers?
2. Why can creation succeed while the Pod is still Pending?
3. What changes when a controller replaces a Pod?

<details>
<summary>Answers</summary>

1. Scheduler; kubelet through the runtime.
2. API acceptance precedes scheduling and startup; placement or setup requirements may be unmet.
3. A new Pod identity and potentially IP, while higher-level Deployment/Service identities can stay stable.

</details>
