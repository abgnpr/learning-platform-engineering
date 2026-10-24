# Kubernetes: start here when you have forgotten

Your refresher for setup, Kubernetes basics, and the commands covered so far. Read this page first; open a topic only when you need its explanation.

## The picture

```text
Fedora → Podman → Minikube node → containerd → application containers

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
| Namespace and DNS | `nginx` means local namespace; `nginx.demo` selects namespace `demo` |
| Requests / limits | Scheduling budget / usage constraints |
| Running / Ready | Process lifecycle / readiness to receive traffic |

## Commands that bring the picture back

Examples use namespace `demo` and application `nginx`. Uppercase names are placeholders. Creation requires the namespace/workload to exist as appropriate; [practice](09-practice-and-revision.md) explains the sequence.

```bash
minikube start                              # Resume local cluster
kubectl config current-context              # Check the target before changing it
kubectl get nodes                           # Is the node Ready?
kubectl get pods,svc -A                      # What exists, and in which namespace?

kubectl get pods -n demo -o wide             # Pod IPs and node placement
kubectl describe pod POD_NAME -n demo        # State, ownership, events
kubectl logs POD_NAME -n demo                # Application output

kubectl scale deployment nginx -n demo --replicas=3
kubectl rollout status deployment/nginx -n demo --timeout=120s
kubectl rollout undo deployment/nginx -n demo # Restore previous Pod template

kubectl describe svc nginx -n demo           # Selector, ports, backends
kubectl get endpointslices -n demo           # Backend addresses/conditions
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
| [11. CI/CD platform backlog](11-ci-cd-backlog.md) | Build a reusable GitLab pipeline from commit to Kubernetes deployment and recovery |

**Writing a manifest from scratch?** Open the [YAML manual](10-kubernetes-yaml-manual.md): common fields, annotated templates, nesting rules, and the references that must match.

**Ready to automate delivery?** Work through the [CI/CD platform backlog](11-ci-cd-backlog.md) with GitLab Self-Managed, an external runner, a private registry, and Minikube.

**Learning API communication?** Open the [API notes in the system-design theatre](../learning-system-design/api/README.md), starting with [REST API interview essentials](../learning-system-design/api/rest/README.md): HTTP methods, status codes, design, caching, security, reliability, recall questions, and curl practice. Other communication topics currently have placeholder folders only.

## Make it stick

**Read → hide → explain → run → predict.** Read one idea, hide the answer, explain it in your own words, then use a command to check a prediction. Revisit missed questions after a few days.

These notes cover the ideas encountered in the Minikube learning material and the existing [PostgreSQL example](postgres/README.md). Podman is the detailed setup; other drivers stay brief. Sections marked **next step** introduce nearby ideas without expanding into a full Kubernetes syllabus. Example IPs and versions are not things to memorize.
