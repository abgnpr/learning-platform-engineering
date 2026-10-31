# 0. Before Kubernetes: processes, containers, and desired state

[Index](README.md) · Next: [Setup](01-linux-tooling-and-minikube.md)

**Goal:** explain what is running before learning the objects that manage it. Prerequisites: Linux, basic terminal navigation, `curl`, and Podman. If tools are missing, use [installation](01-linux-tooling-and-minikube.md#install-and-verify-the-tools), then return here. No Kubernetes cluster is needed yet.

## Start with a process and a listener

A program becomes a **process** when executed. It has a PID, memory, open files, and an exit status. A web server accepts connections through a **listening socket**: a protocol, address, and port. nginx normally listens for HTTP over TCP on container port 80.

An IP identifies a network destination; a port identifies a listener there. `localhost` means the current network environment. Inside a Pod it refers to that Pod, not your laptop or a different Pod. A listener bound only to `127.0.0.1` cannot normally accept connections arriving at the Pod IP.

HTTP is an application protocol carried over the connection. DNS resolves names into addresses; it does not send the HTTP request. Successful resolution does not establish that a server is listening or serving correct responses.

## Image, container, registry, and volume

| Term | Meaning | Boundary |
| --- | --- | --- |
| Image | Packaged filesystem layers and startup metadata | Not a running process |
| Registry | Stores and distributes images | Does not run your application |
| Container | Isolated process execution using an image and runtime settings | Shares the host kernel on this Linux setup |
| Tag | Readable image reference, such as `nginx:alpine` | Can move to different bytes |
| Digest | Content identity, such as `repository@sha256:…` | Does not establish application correctness |
| Volume | Storage mounted into a container | Has its own lifetime; is not automatically a backup |

Linux namespaces isolate views such as process IDs and networking. Linux cgroups account for and constrain resources. A **Kubernetes namespace** instead scopes API resource names; it is a different use of the word.

Image layers are read-only. A container normally adds a writable layer. Stop/start of the same container retains that layer; remove/recreate does not. Explicit storage mounts have separate lifetimes.

## Lab: run, inspect, remove, replace

Run on the host as your normal user: this is **rootless Podman**. The next chapter separately explains a rootful Minikube node. Port 8080 must be free and `foundation-web` must be a name reserved for this exercise.

**Predict:** what will `curl` reach, where will nginx log, and will a container-local file appear in a replacement?

```bash
podman run -d --name foundation-web \
  -p 127.0.0.1:8080:80 docker.io/library/nginx:alpine
podman ps
curl --fail --max-time 5 http://127.0.0.1:8080
podman top foundation-web
podman logs foundation-web
podman exec foundation-web sh -c 'echo container-local > /tmp/lesson.txt'
podman exec foundation-web cat /tmp/lesson.txt
```

The publication maps **host loopback port 8080 → container port 80**. Expect an HTML response, nginx processes, and an access-log entry. `sh -c` writes inside the container, not into the host's `/tmp`.

```bash
podman stop foundation-web
podman start foundation-web
podman exec foundation-web cat /tmp/lesson.txt
podman rm -f foundation-web
podman run -d --name foundation-web \
  -p 127.0.0.1:8080:80 docker.io/library/nginx:alpine
podman exec foundation-web cat /tmp/lesson.txt
```

**Observe:** the file survives stop/start. After remove/recreate, `cat` reports it missing and exits nonzero. The same name does not mean the same container. The default web page exists in both because it comes from the image.

**If it fails:** inspect pull errors for registry/network problems. For an occupied port, choose another host port and update the URL. Inspect `podman ps -a` before removing a name collision. If HTTP fails, compare container state, logs, and published ports.

**Cleanup:** `podman rm -f foundation-web` removes this exercise container; it leaves the image cached.

## Why introduce Kubernetes?

Without an orchestrator, someone must notice crashes, start replacements, choose machines, add instances, update them, and give clients a stable destination as instance addresses change. Kubernetes provides APIs and controllers for declaring and maintaining that intent across nodes.

It cannot fix application bugs, manufacture spare capacity, or make independent database writers safe. A local single-node cluster also cannot survive the loss of its only machine.

## Desired state is a repeated comparison

```text
Desired: 3 replicas → observe: 2 → request another Pod → observe again
```

This repeated correction is **reconciliation**. Controllers observe and submit changes through the API. The scheduler and kubelet perform their parts afterward. Operations take time and can fail; API acceptance is one step toward the result.

You will use a **Pod** as the replaceable execution unit, a **Deployment** to manage application revisions and replica intent, and a **Service** as a stable destination. The next chapters let you prove those relationships.

## Checkpoint without notes

1. Explain image versus container using the replacement experiment.
2. Draw the address/port path from host `curl` to nginx.
3. Why does declaring a port not create a listener?
4. Desired replicas is three and one managed Pod disappears: what happens? What if you change desired replicas to two instead?
5. Name an application responsibility Kubernetes cannot take over.

<details>
<summary>Reasoning cues, after attempting the checkpoint</summary>

1. Image = packaged input; container = separate execution with a writable layer.
2. Host `127.0.0.1:8080` → port publication → container TCP port 80 → nginx.
3. The process must actually bind a socket and listen.
4. Restore three in the first case; converge to two in the second.
5. Correct responses, compatible data migrations, and backups are examples.

</details>

Sources: [Podman run](https://docs.podman.io/en/latest/markdown/podman-run.1.html), [Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/).
