# Kubernetes: learn it, explain it, prove it

Build a first foundation using a local Minikube cluster: run an application, understand its controllers, connect to it, configure it, check its health, preserve data, and recover from failures. The concise reference sections remain useful when you return later.

## First-time learning route

Follow this route rather than filename order. New chapters retain the original file numbering so existing links remain usable.

| Order | Lesson | Evidence of understanding |
| --- | --- | --- |
| 1 | [0. Foundations](00-foundations.md) | Explain processes/images/containers; observe replacement and local file loss |
| 2 | [1. Setup](01-linux-tooling-and-minikube.md) | Ready node, intended context, actual runtime |
| 3 | [2. Architecture](02-kubernetes-architecture.md) | Trace an API request and the reconciliation loops |
| 4 | [3. Workloads](03-workloads-and-rollouts.md) | Replace a Pod, scale, break a revision, recover |
| 5 | [4. Networking](04-services-and-networking.md) | Trace a request and repair a selector mismatch |
| 6 | [5. DNS](05-dns-and-namespaces.md) | Demonstrate caller-relative names from two namespaces |
| 7 | [6. Resources](06-resources-and-cgroups.md) | Explain an unschedulable Pod using requests and allocatable |
| 8 | [12. Configuration and health](12-configuration-and-health.md) | Observe config refresh and readiness versus restart behavior |
| 9 | [13. Storage and state](13-storage-and-state.md) and [PostgreSQL lab](postgres/README.md) | Write a row, replace a Pod, restore a backup into another database |
| 10 | [14. Security and access](14-security-and-access.md) | Prove allowed and denied API actions; inspect a restricted container |
| 11 | [7. Troubleshooting](07-troubleshooting.md) | Distinguish competing explanations using evidence |
| 12 | [8. Mastery checks](08-interview-preparation.md) and [9. Independent practice](09-practice-and-revision.md) | Build and recover from requirements without copying a solution |

Use the [YAML manual](10-kubernetes-yaml-manual.md) alongside lessons whenever you need to write a field. Detailed cgroup inspection and the extended kind catalog are second-pass material.

## Lab contract

- Run repository commands from this directory. Shell blocks run on the **host**; `kubectl exec` runs the command after `--` **inside a container**. Use another terminal while a watch or port-forward is running.
- New to the CLI? Read the [kubectl command pattern and options](09-practice-and-revision.md#read-kubectl-commands-once-then-recognize-the-pattern) before setup. Reference command blocks illustrate choices; the sections marked **Lab** provide the intended exercise sequence.
- Guided labs use namespace **`foundation`** and Deployment/container **`nginx`**. Reserve that namespace for these exercises. PostgreSQL uses **`db-lab`**; independent practice uses **`revision-lab`**.
- Check `kubectl config current-context` before changes. Use your local Minikube cluster. [Setup](01-linux-tooling-and-minikube.md) covers prerequisites and installation.
- Apply [lab files](labs/kubernetes/README.md) **by stage**, not as one directory. An earlier stage can reset later settings; recover using the latest completed stage.
- Images are teaching inputs. `nginx:alpine` and `postgres:15` are mutable tags; a release tag such as `busybox:1.37.0` is not a digest guarantee either. Record resolved image IDs as explained in the lab guide. Pin tested digests when exact reproduction matters.
- A Ready node does not prove registry pulls, DNS, or dynamic storage work. Check the dependencies in each lesson. Timeouts prompt investigation.

## How to study

Keep four notebook entries: **prediction → observation → explanation → remaining question**. Record names/UIDs, relevant conditions, and the command supporting your explanation. Outputs vary; relationships matter more than sample IPs.

A lesson is complete when you can explain the problem, predict a change, produce evidence, recover from its failure, and repeat the checkpoint without the solution. Revisit missed checkpoints after a few days and again after a week.

## The picture

```text
Fedora → Podman → Minikube node → configured runtime → application containers

Deployment → ReplicaSet → Pods        Keep the application running
Service → selector → Pod endpoints    Reach whichever Pods are eligible
DNS name → Service address            Find the application by name
Namespace                            Scope its resource names
Requests / limits / cgroups           Plan and constrain resources
```

**The central idea:** declare a desired state; Kubernetes keeps working to make the actual state match.

## Ten things worth remembering

| Idea | Remember it this way |
| --- | --- |
| Minikube / kubectl | Manage the local cluster / talk to its Kubernetes API |
| Pod | Replaceable execution unit containing one or more containers |
| Deployment → ReplicaSet → Pod | Update strategy → replica count → running application |
| Labels / selectors | Attach identifying tags / match those tags |
| Service | Stable destination in front of changing Pods |
| Node IP / Pod IP / ClusterIP | One node / one Pod / one Service |
| port → targetPort | Service entry port → application destination port |
| Namespace and DNS | `nginx` means local namespace; `nginx.foundation` selects namespace `foundation` |
| Requests / limits | Scheduling budget / usage constraints |
| Running / Ready | Process lifecycle / readiness to receive traffic |

## Commands that bring the picture back

Examples use namespace `foundation` and application `nginx`. Uppercase names are placeholders. Create the workload using the [workloads lab](03-workloads-and-rollouts.md#lab-watch-reconciliation-and-recover-a-failed-revision) first.

```bash
minikube start                              # Resume local cluster
kubectl config current-context              # Check the target before changing it
kubectl get nodes                           # Is the node Ready?
kubectl get pods,svc -A                      # What exists, and in which namespace?

kubectl get pods -n foundation -o wide             # Pod IPs and node placement
kubectl describe pod POD_NAME -n foundation        # State, ownership, events
kubectl logs POD_NAME -n foundation                # Application output

kubectl scale deployment nginx -n foundation --replicas=3
kubectl rollout status deployment/nginx -n foundation --timeout=120s
kubectl rollout undo deployment/nginx -n foundation # Restore previous Pod template

kubectl describe svc nginx -n foundation           # Selector, ports, backends
kubectl get endpointslices -n foundation           # Backend addresses/conditions
minikube stop                               # Stop while retaining cluster state
```

## When something fails

**Right cluster? → right namespace? → ready Pods? → matching selector/backends? → correct DNS name and ports? → application logs?**

Deletion is not routine shutdown. Rollback restores a Pod template, not database contents. Three replicas on one laptop still share one laptop failure. Saved Minikube settings and actual container limits are different layers.

## Open only the topic you need

| Topic | Use it to remember |
| --- | --- |
| [1. Setup](01-linux-tooling-and-minikube.md) | Tools, Podman layers, lifecycle, PATH, versions |
| [2. Architecture](02-kubernetes-architecture.md) | Who accepts, stores, schedules, and runs |
| [3. Workloads](03-workloads-and-rollouts.md) | Ownership, labels, YAML, scaling, updates, undo |
| [4. Networking](04-services-and-networking.md) | IPs, ports, Services, EndpointSlices |
| [5. DNS and namespaces](05-dns-and-namespaces.md) | Short names, cross-namespace calls, scope |
| [6. Resources](06-resources-and-cgroups.md) | Units, requests/limits, capacity, cgroups |
| [7. Troubleshooting](07-troubleshooting.md) | Symptom → useful next command |
| [8. Recall questions](08-interview-preparation.md) | Explain it without seeing the answer |
| [9. Command sheet and practice](09-practice-and-revision.md) | Find a command or rehearse the concepts |
| [10. YAML writing manual](10-kubernetes-yaml-manual.md) | Choose a kind, copy a template, connect fields, and validate |
| [12. Configuration and health](12-configuration-and-health.md) | ConfigMap/Secret refresh, startup/readiness/liveness |
| [13. Storage and state](13-storage-and-state.md) | Container files, emptyDir, PVC/PV, database recovery |
| [14. Security and access](14-security-and-access.md) | Service accounts, RBAC, process permissions, network policy |

**Writing a manifest from scratch?** Open the [YAML manual](10-kubernetes-yaml-manual.md): common fields, annotated templates, nesting rules, and the references that must match.

**After the Kubernetes checkpoints:** the separate [CI/CD platform backlog](11-ci-cd-backlog.md) develops automated delivery. It is outside the Kubernetes foundation route.

**Learning API communication?** Open the [API notes in the system-design theatre](../learning-system-design/api/README.md), starting with [REST API interview essentials](../learning-system-design/api/rest/README.md): HTTP methods, status codes, design, caching, security, reliability, recall questions, and curl practice. Other communication topics currently have placeholder folders only.

## Make it stick

**Read → predict → run → observe → explain → recall.** Write the prediction before running the command. After observing, explain the result in your own words, then repeat the checkpoint without looking at its solution.

Completion establishes a practical foundation for ordinary local workloads. Production administration, multi-node failure recovery, advanced scheduling, autoscaling, Ingress/Gateway routing and TLS, observability platforms, and production database operation need further study. See the [independent capstone](09-practice-and-revision.md#independent-capstone) for the completion standard and next steps. Upstream companion: [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/).
