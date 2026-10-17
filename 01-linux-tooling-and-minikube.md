# 1. Setup: what runs where?

[Index](README.md) · Next: [Architecture](02-kubernetes-architecture.md)

## Remember these four things

- **Minikube manages the local cluster. kubectl talks to Kubernetes.**
- **Podman creates your node environment; containerd runs its workload containers.**
- **start resumes, stop retains, delete destroys.**
- **Defaults, profile settings, and enforced limits are different layers.**

## Put the tools in place

```text
Fedora host
└── Podman container: minikube        ← Kubernetes node environment
    ├── Kubernetes components
    └── containerd                   ← workload runtime
        └── application containers in Pods
```

| Tool | Job |
| --- | --- |
| Podman | Provides the container used as a node |
| Minikube | Creates, starts, stops, and configures the local cluster |
| kubectl | Sends requests to the Kubernetes API |
| containerd inside the node | Creates and runs workload containers |

**Driver ≠ runtime.** One creates the node environment; the other runs application containers inside it. On Linux, this node container shares the host kernel, even if its userspace comes from another distribution.

## Everyday lifecycle

```bash
minikube start --driver=podman --cpus=4 --memory=6g # Create/start with intended sizing
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

Your setup is **rootful**. Plain `podman ps` inspects a separate rootless store and may show nothing. Linux can reclaim cache, so low free memory need not mean low available memory.

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
