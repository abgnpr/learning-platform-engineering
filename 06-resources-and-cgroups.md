# 6. Resources and cgroups

[Index](README.md) · Previous: [DNS](05-dns-and-namespaces.md) · Next: [Troubleshooting](07-troubleshooting.md)

## Remember these four things

- **Request = what scheduling plans for. Limit = a usage constraint.**
- **CPU pressure typically throttles. Memory pressure can kill.**
- **Configured budget ≠ advertised capacity ≠ measured usage.**
- **cgroups are Linux resource controls underneath containers.**

## Decode the units first

| Value | Meaning |
| --- | --- |
| `1` CPU | One CPU's worth of execution time |
| `1000m` CPU | Same as 1 CPU; `m` means milli |
| `250m` CPU | 0.25 CPU |
| `256Mi` memory | 256 × 1024² bytes |
| `1Gi` memory | 1024³ bytes |

CPU time is different from owning a particular physical core. `500m` describes half a CPU's worth of time, not “use core number 0.5.”

## Requests and limits on a container

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Read it as: **“Schedule with 0.25 CPU and 256 MiB in mind. Constrain CPU to 0.5 CPU and memory to 512 MiB.”**

| Setting | Used for |
| --- | --- |
| Request | Scheduling fit against node allocatable and other requests; CPU allocation under contention |
| CPU limit | Restricting CPU bandwidth; exceeding demand typically leads to throttling |
| Memory limit | Memory boundary; unreclaimable excess can lead to an OOM kill |

Requests are not live measurements. They are also not an exclusive core assignment. A container may use more than its request when resources are available and its limits allow it.

**Next-step command:** apply settings to an existing nginx Deployment in your practice namespace:

```bash
kubectl set resources deployment/nginx -n demo -c nginx \
  --requests=cpu=250m,memory=256Mi --limits=cpu=500m,memory=512Mi
kubectl rollout status deployment/nginx -n demo --timeout=120s
```

This changes the Pod template and triggers a rollout. These are example values, not measured application sizing.

## Four numbers that answer different questions

| View | Question | Command |
| --- | --- | --- |
| Minikube configuration | What budget did I ask for? | `minikube config view` plus the profile configuration |
| Capacity / allocatable | What does Kubernetes advertise / offer for Pods? | `kubectl describe node minikube` |
| Requests / limits | What was declared for workloads? | `kubectl describe pod POD_NAME -n demo` |
| Actual usage | What is being consumed now? | `kubectl top nodes` / `kubectl top pods -n demo` |

`top` needs a working metrics API, usually metrics-server. The allocated-resource table in `describe node` sums requests/limits; it is not live utilization.

Allocatable is node capacity after reservations. With container-based Minikube, the outer container can impose a tighter budget than host-looking node capacity suggests. Both workloads and Kubernetes components consume that outer budget.

## cgroups: inspect the enforcement layer

Your rootful Podman setup has another container boundary around the whole node:

```text
Host resources
└── Minikube node container: outer resource constraints
    └── Kubernetes workloads: individual resource requests/limits
```

```bash
sudo podman inspect minikube \
  --format 'Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}'

# Requires the node to be running; these are cgroup v2 paths
minikube ssh -- 'cat /sys/fs/cgroup/memory.max'
minikube ssh -- 'cat /sys/fs/cgroup/cpu.max'
minikube ssh -- 'cat /sys/fs/cgroup/cpuset.cpus.effective'
```

| Example output | Read it as |
| --- | --- |
| `memory.max = 6442450944` | Configured memory maximum of 6 GiB |
| `cpu.max = 400000 100000` | Quota/period permits four CPUs' worth of execution time |
| `cpu.max = max 100000` | No CPU bandwidth quota at this cgroup |
| `cpuset.cpus.effective = 0-3` | Effective CPU set includes CPUs 0 through 3 |

**No quota does not mean no restrictions.** CPU sets, affinity, parent cgroups, and contention still matter. `NanoCPUs=0` alone does not describe every CPU control. A configured memory maximum is not proof that you stress-tested enforcement.

<details>
<summary>QoS: recognize the names for now</summary>

`BestEffort`, `Burstable`, and `Guaranteed` classify Pods based on resource settings. The nginx example without CPU/memory requests or limits is BestEffort; it still uses real resources. Detailed QoS and eviction behavior are next-step study.

</details>

## Test your recall

1. Is `250m` one quarter of a CPU or 250 CPUs?
2. Does low actual usage guarantee another Pod can be scheduled?
3. Does `max 100000` prove access to every CPU on the laptop?

<details>
<summary>Answers</summary>

1. One quarter of a CPU.
2. No. Scheduling considers requests, allocatable resources, and other placement constraints.
3. No. It only describes bandwidth quota at that cgroup; other controls may restrict execution.

</details>

References: [Kubernetes resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), [Linux cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html).
