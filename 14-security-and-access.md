# 14. Security and access: identity, permissions, and boundaries

[Index](README.md) · Learning route: [Storage](13-storage-and-state.md) → this lesson → [Troubleshooting](07-troubleshooting.md)

**Goal:** distinguish API permissions, process privileges, and network access, then produce evidence for each relevant boundary. Prerequisites: the guided `foundation` namespace and configuration lesson. This is a local-cluster exercise using your existing administrator context; do not change real users' access.

## Ask which boundary controls the action

| Question | Mechanism |
| --- | --- |
| Who is calling the Kubernetes API? | Authentication: user credentials or a ServiceAccount token |
| May that identity perform this API operation? | Authorization, commonly RBAC |
| Which Linux user and privileges does this container process have? | Pod/container `securityContext` and runtime enforcement |
| Can one workload send traffic to another? | Networking and enforced NetworkPolicy |
| Is this application request authorized? | Application authentication/authorization |

A namespace organizes resources but is not sufficient isolation. RBAC permission to read Pods is different from TCP permission to call their application. Running as non-root is different from having a restricted Kubernetes API identity.

## Lab A: grant one identity limited API access

A **ServiceAccount** is a namespaced workload identity. A **Role** lists allowed API groups/resources/verbs in a namespace; a **RoleBinding** gives that Role to an identity. RBAC grants are additive: there is no explicit deny rule that overrides another binding's allow.

[access.yaml](labs/kubernetes/access.yaml) allows `get` and `list` on ConfigMaps in `foundation`. It grants no Secret reads, writes, or cross-namespace access.

**Predict:** which checks below should return yes or no?

```bash
kubectl apply -f labs/kubernetes/access.yaml
kubectl auth can-i get configmaps -n foundation \
  --as=system:serviceaccount:foundation:config-reader
kubectl auth can-i get secrets -n foundation \
  --as=system:serviceaccount:foundation:config-reader
kubectl auth can-i create configmaps -n foundation \
  --as=system:serviceaccount:foundation:config-reader
kubectl auth can-i get configmaps -n kube-system \
  --as=system:serviceaccount:foundation:config-reader
kubectl get configmap nginx-content -n foundation \
  --as=system:serviceaccount:foundation:config-reader
```

Expect **yes, no, no, no**, then a permitted ConfigMap read. A `no` response may return a nonzero exit status; that is expected evidence here. Try an actual denied read with `kubectl get secrets -n foundation --as=system:serviceaccount:foundation:config-reader`; expect Forbidden without Secret values.

`--as` uses **impersonation**, requiring the current caller to have permission to impersonate. It is a local-admin testing technique; it does not mount a token or prove in-Pod token delivery. If impersonation itself is forbidden, distinguish that from the tested identity's access. Unexpected permission grants require inspection of existing RoleBindings/ClusterRoleBindings. [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

The teaching HTTP app does not call the Kubernetes API, so its Pod sets `automountServiceAccountToken: false`. The kubelet can still inject referenced ConfigMap/Secret values; the application does not need direct API read permission for that injection. If an application needs API access, deliberately choose its service account, grants, and token delivery.

## Lab B: restrict a container's operating-system privileges

Read [security-demo.yaml](labs/kubernetes/security-demo.yaml). It uses a numeric non-root UID, prevents privilege escalation, drops capabilities, uses the runtime's default seccomp profile, and makes the image filesystem read-only. A separate `emptyDir` supplies writable `/tmp`.

```bash
kubectl apply -f labs/kubernetes/security-demo.yaml
kubectl wait pod/security-demo -n foundation --for=condition=Ready --timeout=120s
kubectl exec -n foundation security-demo -- id
kubectl exec -n foundation security-demo -- touch /should-fail
kubectl exec -n foundation security-demo -- touch /tmp/works
kubectl exec -n foundation security-demo -- ls -l /tmp/works
kubectl exec -n foundation security-demo -- ls /var/run/secrets/kubernetes.io/serviceaccount
```

Expect UID 10000, a failed root-filesystem write, a successful `/tmp` write, and no automatically mounted service-account directory. The write failure alone can also reflect ordinary Unix permissions; inspect the declared `readOnlyRootFilesystem` field to establish the intended configuration. A missing token directory supports the automount check, not a claim that no other credentials exist anywhere.

These settings are teaching controls, not a promise that an arbitrary application works unchanged under them. For example, images that require writable startup directories must be adapted. Cluster **Pod Security Admission** can enforce standards; setting a securityContext on one Pod is not cluster-wide enforcement. [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

## NetworkPolicy: only as strong as its enforcement

By default, workloads are generally non-isolated for ingress and egress unless policies select them. A NetworkPolicy is enforced by a supporting network implementation. The API accepting its YAML is not proof that packets are blocked; the default local networking may not support the intended enforcement.

For a later isolation experiment, first identify/configure a policy-capable CNI using its documentation. In a disposable namespace, prove allowed traffic, apply a deny policy, prove new connections fail, then add a narrow allow and prove only that path returns. Include DNS egress when restricting name-based clients. Record source/destination labels and namespace selectors. No deny policy is silently applied by this lesson. [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

## Recovery, cleanup, and checkpoint

Delete the exact exercise resources when finished:

```bash
kubectl delete -f labs/kubernetes/access.yaml
kubectl delete pod security-demo -n foundation
```

If a Pod cannot start, inspect admission errors, image identity/permissions, and events. Recreate only the teaching Pod after fixing its file if an immutable field needs changing.

**Checkpoint:** show one allowed API read and one denied operation; explain why that says nothing about nginx HTTP reachability. Explain why a read-only root filesystem still permits the mounted `/tmp` write. Describe what extra evidence would be required to claim NetworkPolicy enforcement. Continue with [troubleshooting](07-troubleshooting.md).
