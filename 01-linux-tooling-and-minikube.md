# 1. Setup: what runs where?

[Index](README.md) · Previous: [Foundations](00-foundations.md) · Next: [Architecture](02-kubernetes-architecture.md)

**Goal:** establish which tool runs at which layer, and verify a usable cluster before studying workloads. Prerequisite: [process/container foundations](00-foundations.md). You can install the tools below before doing that chapter's container lab.

## Install and verify the tools

Use a Linux host with internet access and enough disk, CPU, and memory for both Kubernetes and applications. Minikube documents a minimum of two CPUs, 2 GB available memory, and 20 GB free disk; the examples request four CPUs and 6 GB for more lab headroom. If that exceeds your host's available resources, reduce the cluster budget and run fewer workloads concurrently.

1. On Fedora, install Podman and curl using `sudo dnf install podman curl`. Check `podman version` and `curl --version`.
2. Install Minikube using the [official Linux installation instructions](https://minikube.sigs.k8s.io/docs/start/), choosing your CPU architecture (`uname -m`). Record the installed release rather than assuming the download remains the same over time.
3. Install kubectl using the [official Linux instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), including the documented checksum check for a downloaded binary. Match its supported version skew to your server. Minikube's bundled client, `minikube kubectl -- …`, is an alternative if you have no standalone kubectl.
4. Run `minikube version` and `kubectl version --client`. Check `type -a minikube kubectl` if installation and execution disagree.

The Podman driver is experimental upstream. Rootful Podman is the existing Fedora path in these notes; the driver guide recommends CRI-O for that path and containerd for rootless Podman. Follow the [driver prerequisites](https://minikube.sigs.k8s.io/docs/drivers/podman/) for sudo/rootless setup. Docker is an alternative supported driver if Podman-specific problems prevent progress.

For a **new** default profile on this rootful path:

```bash
minikube start --driver=podman --container-runtime=cri-o --cpus=4 --memory=6g
minikube status
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" runtime="}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'
kubectl get pods -n kube-system
```

**Expected evidence:** context `minikube`, a node reporting `Ready`, a reported runtime, and system Pods becoming ready. Initial image pulls take time. `minikube start` provisions infrastructure; `kubectl get` observes API objects. If the existing profile uses containerd or another driver, resume it with `minikube start` and inspect it; changing defaults does not migrate it. Both CRI-O and containerd implement the runtime role below.

**If it fails:** read Minikube's error, run `minikube status` and `minikube logs --problems`, then inspect host resources and the selected driver. An unreachable API must be resolved before application labs. An existing context for a different cluster must not be used for these exercises.

**Checkpoint:** stop with `minikube stop`, resume with `minikube start`, and reproduce the Ready-node/runtime evidence. Explain why stopping retains state and why it is not a backup. Record client, server, Minikube, driver, and runtime versions in your notebook. Do not delete the profile as part of this checkpoint.

## Remember these four things

- **Minikube manages the local cluster. kubectl talks to Kubernetes.**
- **Podman creates your node environment; CRI-O or containerd runs its workload containers.**
- **start resumes, stop retains, delete destroys.**
- **Defaults, profile settings, and enforced limits are different layers.**

## Put the tools in place

```text
Fedora host
└── Podman container: minikube        ← Kubernetes node environment
    ├── Kubernetes components
    └── CRI-O / containerd           ← configured workload runtime
        └── application containers in Pods
```

| Tool | Job |
| --- | --- |
| Podman | Provides the container used as a node |
| Minikube | Creates, starts, stops, and configures the local cluster |
| kubectl | Sends requests to the Kubernetes API |
| CRI-O / containerd inside the node | Creates and runs workload containers |

**Driver ≠ runtime.** One creates the node environment; the other runs application containers inside it. On Linux, this node container shares the host kernel, even if its userspace comes from another distribution.

## Everyday lifecycle

```bash
minikube start --driver=podman --container-runtime=cri-o --cpus=4 --memory=6g
minikube start                  # Resume an existing cluster
minikube status                 # Check its state
minikube stop                   # Stop while keeping state on disk
minikube profile list           # List separate local cluster profiles
minikube ssh                    # Enter the node environment
```

A **profile** identifies a separate Minikube cluster. Add `-p PROFILE` when you need to select one explicitly.

`minikube delete -p PROFILE` intentionally destroys that profile and its cluster-local state. Use stop for everyday shutdown; retaining state is still not a backup.

## Save preferences; verify enforcement separately

```bash
minikube config set driver podman
minikube config set cpus 4
minikube config set memory 6g
minikube config view
```

| Layer | Meaning |
| --- | --- |
| Global defaults | Preferences for cluster creation |
| Profile configuration | Settings recorded for an existing cluster |
| Runtime/cgroups | Constraints actually configured on its node container |

Start flags do not automatically become global defaults. Saving defaults does not resize a running node. Follow the driver's supported procedure and recreation messages; see [configuration](https://minikube.sigs.k8s.io/docs/commands/config/).

## Check host resources and the right Podman store

```bash
uname -m                      # Architecture for choosing binaries
nproc                         # Available logical processors
free -h                       # Inspect available memory, not just free memory
sudo podman ps -a              # Rootful containers, including stopped ones
sudo podman inspect minikube   # Node container settings and state
```

The node setup above is **rootful**. Plain `podman ps` inspects a separate rootless store, including the container from lesson 0, and may show no node. Linux can reclaim cache, so low free memory need not mean low available memory.

## When an update seems ineffective

<details>
<summary>Open PATH, binary selection, and version checks</summary>

```bash
type -a minikube               # Paths, aliases, functions
command -v minikube            # Selected command
minikube version
kubectl version --client
kubectl version               # API server version too, if reachable
kubectl get nodes             # VERSION here is node kubelet version
minikube kubectl -- get nodes  # Minikube-managed kubectl
```

**Installed ≠ selected.** Check absolute-path versions if several binaries exist. `rpm -qf /path/to/binary` identifies Fedora package ownership. After changing paths, use `rehash` in zsh or `hash -r` in bash if command lookup is stale.

Minikube, Kubernetes, and kubectl have independent versions. Updating a CLI does not upgrade an existing cluster. kubectl is supported within one minor version of the API server; see [compatibility](https://kubernetes.io/releases/version-skew-policy/).

Missing command? `dnf provides '*/COMMAND'` discovers the Fedora package provider.

</details>

## Other options — good to know

| Driver | Idea |
| --- | --- |
| Podman — your path | Container-based node on Linux |
| Docker | Another container-based driver |
| KVM2/libvirt | VM-based Linux driver needing hardware virtualization |

References: [Podman driver](https://minikube.sigs.k8s.io/docs/drivers/podman/), [KVM2 option](https://minikube.sigs.k8s.io/docs/drivers/kvm2/). The Podman guide marks the driver experimental.

## Test your recall

1. Which tool resumes the cluster? Which lists its Pods?
2. Why might plain `podman ps` show no node?
3. Does saved `cpus: 4` prove a running node has a four-CPU quota?

<details>
<summary>Answers</summary>

1. `minikube start`; `kubectl get pods`.
2. Wrong rootless/rootful store, or a stopped container requiring `ps -a`.
3. No. Check profile and runtime controls separately.

</details>
