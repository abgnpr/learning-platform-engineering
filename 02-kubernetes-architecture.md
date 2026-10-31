# 2. Architecture: who does what?

[Index](README.md) · Previous: [Setup](01-linux-tooling-and-minikube.md) · Next: [Workloads](03-workloads-and-rollouts.md)

## Remember these four things

- **API accepts → etcd stores → controllers reconcile.**
- **Scheduler chooses where; kubelet makes it run there through the runtime.**
- **Cluster contains nodes; nodes run Pods; Pods contain containers.**
- **Running, Ready, and schedulable answer different questions.**

**Goal:** explain the chain of responsibility, including where progress can stop. Prerequisite: a Ready node from setup. This lesson observes the cluster; the next creates the application.

## Why these layers exist

The **control plane** maintains cluster intent and makes decisions. A **node** supplies execution resources; its kubelet acts on assigned Pods. Separating these responsibilities lets one API describe work that may run on different machines. In single-node Minikube the roles share a machine, but the responsibilities still differ.

A **Pod** gives one or more tightly coupled containers a shared network identity and access to declared volumes. Containers in one Pod share IP/port space and can communicate using localhost; they must avoid conflicting listeners. Replicas of a web app normally go in separate Pods so they can be placed and replaced independently. A Pod is an execution unit, not a smaller virtual machine or a durable identity for your application.

Objects such as Deployments and Services are records in the API. They have a kind, name, namespace where applicable, and UID. Recreating an object with the same name gives a new UID. `spec` usually describes intent; `status` reports observations. Controllers reconcile over time, so the two can temporarily disagree.

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
| CRI-O / containerd runtime | Creates and runs the containers |
| CNI networking | Configures Pod networking |
| kube-proxy | Programs Service forwarding in this setup |
| CoreDNS | Resolves cluster names |
| Storage provisioner | Supplies volumes for supported storage requests |

The first four form the core control plane. Some networking implementations replace kube-proxy. See [architecture](https://kubernetes.io/docs/concepts/architecture/).

etcd stores Kubernetes API data, including object configuration and Secret objects. It does not store your image registry or the PostgreSQL files mounted from a volume. Backing up cluster objects and backing up application data are separate responsibilities.

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

**Observe now:** run the first two commands below. Pick one system Pod, then run `kubectl describe pod SYSTEM_POD -n kube-system` with its actual name. Locate its node, container image, conditions, and events. Application commands below become usable after chapter 3.

```bash
kubectl get nodes -o wide           # Nodes, health, addresses, kubelet versions
kubectl get pods -n kube-system     # System Pods
kubectl get pods -n foundation -o wide    # App Pods, IPs, and node placement
kubectl describe pod POD_NAME -n foundation # State, owner, conditions, events
kubectl logs POD_NAME -n foundation       # Application output
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

You will observe both in [configuration and health](12-configuration-and-health.md). A `Running` phase can coexist with a container waiting/restarting or a Pod being unready. `CrashLoopBackOff` and `ImagePullBackOff` in kubectl's STATUS column are diagnostic summaries, not additional Pod phases.

A control-plane role label alone does not block workloads; taints and scheduling rules can. Three replicas on one laptop share one physical failure domain.

A Pod's service account is its API identity. Authorization such as RBAC determines what that identity may do.

## Checkpoint: place the failure in the chain

Before checking answers, identify who still has work to do:

| Observation | Investigation |
| --- | --- |
| API says created; Pod has no node and event says insufficient CPU | Scheduler cannot find a fit; inspect requests and node allocatable |
| Pod has a node; image pull is denied | Kubelet/runtime cannot obtain the image; inspect reference and registry access |
| Container started; HTTP fails | Inspect readiness, listener, request path, and application logs |

The API response proves acceptance, not successful scheduling or startup. After chapter 3, return and trace one actual Pod through its owner references, node assignment, and events. Event retention is finite; absence of an old event does not prove it never occurred.

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
