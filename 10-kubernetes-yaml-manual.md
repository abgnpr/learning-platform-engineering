# 10. Kubernetes YAML: a manual for writing from scratch

This is the field reference for the [learning route](README.md), not a first-pass syllabus. For executable staged manifests use the [lab guide](labs/kubernetes/README.md). Authoring YAML expresses intent; only runtime checks establish that images, references, probes, and storage actually work.

[Index](README.md) · Related: [Workloads](03-workloads-and-rollouts.md), [Networking](04-services-and-networking.md), [Resources](06-resources-and-cgroups.md)

## Start with these five rules

1. **`kind` decides what you are creating.** The filename is your choice; `deployment.yaml` is a convention, not a special file type.
2. **`apiVersion` + `kind` decide the schema.** Different objects have different fields.
3. **`metadata` describes this object.** A template's metadata describes the Pods it will create.
4. **`spec` expresses desired behavior where that kind uses it.** ConfigMap and Secret use `data`/`stringData`; Namespace needs no `spec` in a basic manifest.
5. **Most mistakes are nesting or mismatched references.** Check where a field belongs and what it points to.

## Choose what you want to create

| I want to… | Kind / usual filename | Jump to |
| --- | --- | --- |
| Organize objects under a namespace | Namespace / `namespace.yaml` | [Namespace](#namespace) |
| Keep an application running and update it | Deployment / `deployment.yaml` | [Deployment](#deployment) |
| Give Pods a stable network destination | Service / `service.yaml` | [Service](#service) |
| Supply ordinary configuration | ConfigMap / `configmap.yaml` | [ConfigMap](#configmap) |
| Supply sensitive configuration | Secret / `secret.yaml` | [Secret](#secret) |
| Request persistent storage | PersistentVolumeClaim / `pvc.yaml` or `storage.yaml` | [PVC](#persistentvolumeclaim) |
| Run a standalone/debug container | Pod / `pod.yaml` | [Pod](#pod) |
| Run work to completion or on a schedule | Job / CronJob | [Jobs](#job-and-cronjob) |
| Find another workload, policy, or infrastructure type | See the short catalog | [Other kinds](#other-kinds-to-recognize) |

One file can contain several objects separated by `---`. Several files can describe one application. Kubernetes reads their object definitions, not the meaning of their filenames. Installed custom resources add more kinds; this is a manual for common kinds, not a fixed list of everything a cluster can support.

## Common fields: learn the outer shape once

| Field | Meaning | What you normally write |
| --- | --- | --- |
| `apiVersion` | API group/version for this kind | `v1`, `apps/v1`, `batch/v1`, etc. |
| `kind` | Object type; case matters | `Deployment`, `Service`, etc. |
| `metadata.name` | This object's name | A descriptive name such as `nginx` |
| `metadata.namespace` | Namespace of a namespaced object | `foundation`; omit for cluster-scoped kinds |
| `metadata.labels` | Tags used for grouping/selection | String keys/values such as `app: nginx` |
| `metadata.annotations` | Extra metadata for people/tools | String values; not the normal selector mechanism |
| `spec` | Kind-specific desired state | Different structure for each kind |
| `status` | Observed state reported by the system | Leave out of a new manifest |

Also leave generated `uid`, `resourceVersion`, `managedFields`, and creation timestamps out of starter manifests. A live `kubectl get ... -o yaml` response is more than a clean authoring template. See [Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/).

## YAML rules that prevent avoidable mistakes

- Use spaces, not tabs. Two spaces per level is a useful convention; sibling indentation must agree.
- `key: value` is a mapping entry. `-` starts an item in a list. `containers`, `env`, `ports`, and `volumeMounts` are lists.
- Quote text that could be parsed as a number or boolean: `value: "8080"`, `APP_ENABLED: "true"`. Environment values, ConfigMap text, labels, and annotations are strings.
- Keep actual numeric fields numeric: `replicas: 3`, `port: 80`. Named ports such as `targetPort: http` are strings by design.
- `|` starts a multiline string, useful for a ConfigMap file. `---` separates complete objects. `#` starts a comment outside a quoted string.
- A duplicate key is a mistake even if a parser silently keeps the last value. Valid YAML syntax alone does not establish a valid Kubernetes object.

## Where does the Pod configuration go?

**Learn the Pod shape once; controllers wrap it.**

| Kind | Path to Pod configuration | Path to containers |
| --- | --- | --- |
| Pod | `spec` | `spec.containers` |
| Deployment, ReplicaSet, StatefulSet, DaemonSet, Job | `spec.template.spec` | `spec.template.spec.containers` |
| CronJob | `spec.jobTemplate.spec.template.spec` | `spec.jobTemplate.spec.template.spec.containers` |

Deployment `metadata.labels` does **not** automatically label its Pods. Put Pod labels in `spec.template.metadata.labels`.

## Templates: open only the one you need

Examples use namespace `foundation`, application `nginx`, backend port 80, and Service port 8080. Each object is a separate manifest you can save under its suggested filename. Create the namespace first. Referenced configuration/storage must exist before the workload can use it. These templates are independent field examples, not successive lab stages; use the [saved lab files](labs/kubernetes/README.md) for the guided application. Image tags and resource values are teaching examples, not measured application sizing.

### Namespace

<details>
<summary>namespace.yaml — create a scope for app resources</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: foundation
```

Namespace is cluster-scoped: do not give it `metadata.namespace`. A namespace does not automatically isolate network traffic.

</details>

### Deployment

<details>
<summary>deployment.yaml — the main application template, annotated</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx                    # Deployment name
  namespace: foundation
  labels:
    app: nginx                   # Labels THIS Deployment
spec:
  replicas: 2                    # Desired Pod count
  selector:
    matchLabels:
      app: nginx                 # Must match the Pod template's labels
  template:                      # Blueprint for Pods
    metadata:
      labels:
        app: nginx               # Labels the PODS; Service matches these
    spec:                        # Pod configuration starts here
      containers:
        - name: nginx            # Container name; used by set image/log selection
          image: nginx:alpine
          ports:
            - name: http
              containerPort: 80  # Describes the port; nginx must actually listen
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
          readinessProbe:        # Does this container pass a readiness check?
            httpGet:
              path: /
              port: http         # Refers to the named container port above
            periodSeconds: 5
```

**Check the three connections:** Deployment selector ↔ template labels; Service selector ↔ template labels; Service targetPort ↔ application port.

| Optional field | Where it goes / what it changes |
| --- | --- |
| `strategy` | Under Deployment `spec`; controls rolling update versus recreate |
| `revisionHistoryLimit` | Under Deployment `spec`; retained old ReplicaSets for revision history |
| `imagePullPolicy` | Under a container; `Always`, `IfNotPresent`, or `Never` |
| `env`, `envFrom` | Under a container; environment configuration |
| `volumeMounts` | Under a container; where Pod volumes appear in it |
| `volumes` | Under Pod configuration; sources of named volumes |

A Deployment's Pod restart policy is `Always`; the default can be omitted. Treat Deployment selectors as a design choice at creation: they are immutable. See [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

</details>

### Service

<details>
<summary>service.yaml — connect clients to matching Pods</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: foundation
spec:
  type: ClusterIP                # Default; internal access
  selector:
    app: nginx                   # Matches POD labels in namespace foundation
  ports:
    - name: http
      protocol: TCP
      port: 8080                 # Client calls Service on this port
      targetPort: http           # Named container port; numeric 80 also works
```

For NodePort, change `type` to `NodePort`. Omit `nodePort` to let the API allocate one; if choosing it explicitly, it must be in the cluster's allowed range and available. `nodePort` belongs inside the `ports` list item, alongside `port` and `targetPort`.

Service name and Deployment name need not match. The **selector** connects them. Clients use port 8080 while nginx listens on target port 80. See [Services](https://kubernetes.io/docs/concepts/services-networking/service/).

</details>

### ConfigMap

<details>
<summary>configmap.yaml — ordinary configuration as strings or files</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: foundation
data:                            # No spec wrapper
  APP_MODE: "development"
  APP_ENABLED: "true"
  app.properties: |              # A multiline file stored under this key
    log.level=info
    worker.count=2
```

Use `env`/`envFrom` to supply values as environment variables, or a ConfigMap-backed volume to supply files. Environment values in existing containers do not refresh just because the ConfigMap changed. See [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/).

</details>

### Secret

<details>
<summary>secret.yaml — sensitive configuration, with a placeholder value</summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: foundation
type: Opaque
stringData:                      # Plain strings converted into data by the API
  API_TOKEN: "replace-locally"   # Illustrative placeholder, not a real credential
```

`data` requires base64-encoded values; `stringData` accepts ordinary strings. Base64 is encoding, not encryption. Keep real credentials out of committed example files. Application containers can reference a Secret key without embedding its value in the Deployment.

`stringData` is convenient for authoring but does not work well with server-side apply; choose an appropriate generated/managed Secret workflow when using that mode. See [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

</details>

### PersistentVolumeClaim

<details>
<summary>pvc.yaml — request backing storage</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: foundation
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # storageClassName: CLASS_NAME # Optional: choose an existing StorageClass
```

Omitting `storageClassName` allows the default StorageClass to be used. An explicitly empty value `""` requests no class, which is different. A provisioner or matching existing PV must satisfy the claim; YAML acceptance does not guarantee it becomes Bound.

`ReadWriteOnce` describes read-write mounting by one node, not necessarily one Pod. A PVC is namespaced; its backing PersistentVolume is cluster-scoped. See [persistent storage](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

</details>

### Pod

<details>
<summary>pod.yaml — a standalone debug workload; compare its nesting</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug
  namespace: foundation
spec:                            # No template wrapper for a standalone Pod
  restartPolicy: Never
  containers:
    - name: debug
      image: busybox:1.36
      command: ["sh", "-c"]
      args: ["sleep 3600"]
```

`command` overrides the image entrypoint; `args` supplies/replaces its arguments. A shell is not implicit: request `sh -c` if you need shell operators or shell variable expansion. Deleting a standalone Pod does not make a Deployment recreate it. Prefer a controller for a long-running application. See [Pods](https://kubernetes.io/docs/concepts/workloads/pods/).

</details>

### Job and CronJob

<details>
<summary>Next step: job.yaml / cronjob.yaml — complete once or on a schedule</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-once
  namespace: foundation
spec:
  backoffLimit: 2                 # Retry budget before Job failure
  template:
    spec:
      restartPolicy: Never       # Jobs allow Never or OnFailure
      containers:
        - name: hello
          image: busybox:1.36
          command: ["sh", "-c", "echo hello"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-scheduled
  namespace: foundation
spec:
  schedule: "*/5 * * * *"        # Every five minutes; quote the expression
  timeZone: Asia/Kolkata
  concurrencyPolicy: Forbid      # Do not overlap Jobs from this CronJob
  jobTemplate:
    spec:                        # Job configuration
      backoffLimit: 2
      template:
        spec:                    # Pod configuration
          restartPolicy: Never
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "echo hello"]
```

Read the wrappers: **CronJob → Job template → Pod template → container**. `completions`, `parallelism`, and `ttlSecondsAfterFinished` are useful Job fields to look up when needed. Scheduled work should tolerate retries/duplicate execution rather than assume exactly-once delivery.

References: [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/), [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

</details>

## Reusable container and volume fields

<details>
<summary>Configuration references: env / envFrom</summary>

This is a **container fragment**, not a complete manifest. Put these fields under the relevant `containers` list item. The ConfigMap and Secret above must exist in the Pod's namespace. The nginx image does not automatically act on these example application variables.

```yaml
env:
  - name: LOG_LEVEL
    value: "info"                # Literal environment value: a string
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config         # ConfigMap metadata.name
        key: APP_MODE            # Exact key under its data
  - name: API_TOKEN
    valueFrom:
      secretKeyRef:
        name: app-secret         # Secret metadata.name
        key: API_TOKEN           # Exact key in the Secret
```

To import all applicable keys, use this alternative container fragment:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

Use explicit `env` entries when you want to choose individual keys or rename them for the application.

</details>

<details>
<summary>Storage wiring: PVC → volume → container mount</summary>

At **Pod configuration level**, alongside `containers`, define the volume source:

```yaml
volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: app-data        # PVC metadata.name, in the same namespace
```

Inside the **container**, define where it appears:

```yaml
volumeMounts:
  - name: data-volume             # Must match volumes[].name above
    mountPath: /data              # Directory inside this container
```

Remember **claimName → PVC name**, **mount name → volume name**, **mountPath → application filesystem**. For a Deployment, `volumes` is under `spec.template.spec`; `volumeMounts` is under a container inside that Pod spec.

The [PostgreSQL Deployment](postgres/deployment.yaml) shows this same pattern with a database data directory and a password Secret reference.

</details>

<details>
<summary>Other common Pod/container fields: find the correct level</summary>

| Field | Level | Purpose |
| --- | --- | --- |
| `containers`, `initContainers` | Pod configuration | Main containers / initialization before application containers |
| `serviceAccountName` | Pod configuration | API identity assigned to the Pod |
| `imagePullSecrets` | Pod configuration | Registry authentication Secret references |
| `nodeSelector`, `affinity`, `tolerations` | Pod configuration | Placement constraints/preferences and taint toleration |
| `restartPolicy` | Pod configuration | Container restart behavior, constrained by the owning kind |
| `terminationGracePeriodSeconds` | Pod configuration | Grace period for termination |
| `securityContext` | Pod or container, with different supported fields | Process/filesystem/security settings |
| `name`, `image`, `imagePullPolicy` | Container | Container identity and image behavior |
| `command`, `args`, `workingDir` | Container | Process launch configuration |
| `ports`, `env`, `envFrom` | Container | Port declarations and environment configuration |
| `resources` | Container in these examples | CPU/memory requests and limits |
| `readinessProbe` | Container | Check readiness for traffic |
| `livenessProbe` | Container | Check whether the container needs restarting |
| `startupProbe` | Container | Allow startup checks before readiness/liveness checks begin |
| `volumeMounts` | Container | Mounts of Pod-defined volumes |

These are lookup cues, not fields to add to every manifest. Use `kubectl explain` for supported subfields, defaults, and required values.

For the health-check fields, see [readiness, liveness, and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

</details>

## Other kinds to recognize

<details>
<summary>Broader catalog — choose the kind, then look up its fields</summary>

These are later-reference types. They are not all necessary for a basic application.

| Kind | Common API version | Purpose / important fields |
| --- | --- | --- |
| ReplicaSet | `apps/v1` | `replicas`, `selector`, `template`; normally created by a Deployment |
| StatefulSet | `apps/v1` | Stable Pod identities/storage; `serviceName`, `selector`, `template`, `volumeClaimTemplates` |
| DaemonSet | `apps/v1` | Pods across eligible nodes; `selector`, `template` |
| PersistentVolume | `v1` | Backing storage; `capacity`, `accessModes`, reclaim policy, storage source such as `csi` |
| StorageClass | `storage.k8s.io/v1` | Provisioning policy; top-level `provisioner`, `parameters`, `volumeBindingMode` |
| Ingress | `networking.k8s.io/v1` | HTTP routing; `ingressClassName`, `rules`, paths/backends, `tls`; needs a controller |
| Gateway / HTTPRoute | `gateway.networking.k8s.io/v1` | Gateway API routing; needs installed API resources and a supporting implementation |
| ServiceAccount | `v1` | Pod API identity; referenced by `serviceAccountName` |
| Role / ClusterRole | `rbac.authorization.k8s.io/v1` | Top-level `rules` with `apiGroups`, `resources`, `verbs`; permissions, not a workload |
| RoleBinding / ClusterRoleBinding | `rbac.authorization.k8s.io/v1` | Top-level `subjects` and `roleRef`; bind identities to permissions |
| NetworkPolicy | `networking.k8s.io/v1` | `podSelector`, `policyTypes`, `ingress`, `egress`; needs enforcement by networking |
| HorizontalPodAutoscaler | `autoscaling/v2` | `scaleTargetRef`, replica bounds, `metrics`; needs metrics support |
| PodDisruptionBudget | `policy/v1` | `selector` plus `minAvailable` or `maxUnavailable`; controls supported voluntary disruptions |

Namespace, PersistentVolume, StorageClass, ClusterRole, and ClusterRoleBinding are cluster-scoped. Most application objects above are namespaced. Not every kind uses `spec`: StorageClass and RBAC objects are further examples.

Field references for these less familiar shapes: [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/), and [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

API availability can vary by cluster/version and installed extensions. Ask the cluster rather than memorizing a supposedly exhaustive list:

```bash
kubectl api-resources
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
kubectl api-versions
```

</details>

## The connections to check before applying

```text
Deployment spec.selector.matchLabels ──matches──→ template.metadata.labels
Service spec.selector ─────────────────matches──→ template.metadata.labels
Service targetPort ─────────────────────────────→ container's listening port/name
env.valueFrom.*KeyRef.name + key ────────────────→ ConfigMap/Secret name + key
volumes[].persistentVolumeClaim.claimName ───────→ PVC metadata.name
container.volumeMounts[].name ──────────────────→ Pod volumes[].name
```

Names must match the correct reference; they do not all need the same text. In particular, a PVC name, a volume name, a Deployment name, and a container name can all differ.

## Write, inspect, validate, apply

1. Choose the kind from the table.
2. Start from its small template and set name/namespace.
3. Put Pod and container fields at the correct nesting level.
4. Check selectors, ports, and references using the connection diagram.
5. Validate against the intended cluster, review the diff, then apply.

### Look up fields instead of guessing

These commands use the cluster's published schema and require API access:

```bash
kubectl explain deployment --api-version=apps/v1
kubectl explain deployment.spec
kubectl explain deployment.spec.template.spec.containers
kubectl explain service.spec.ports
kubectl explain pvc.spec
kubectl explain deployment.spec --recursive
```

`explain` answers **“what fields belong here?”**; `get -o yaml` answers **“what does this existing object contain?”** See [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/).

### Generate a starter when a command supports it

```bash
kubectl create deployment nginx -n foundation --image=nginx:alpine \
  --dry-run=client -o yaml > deployment.yaml
kubectl create namespace foundation --dry-run=client -o yaml > namespace.yaml
```

These commands write local files and overwrite an existing file at the same path. They do not create those objects in the cluster. Review and edit the generated manifest.

### Check against the API before saving changes there

Assume namespace `foundation` already exists and the referenced files contain your intended objects:

```bash
kubectl config current-context
kubectl apply --dry-run=server --validate=strict -f deployment.yaml
kubectl diff -f deployment.yaml
kubectl apply -f deployment.yaml
kubectl rollout status deployment/nginx -n foundation --timeout=120s
```

Server dry-run validates without persisting the resource. It needs the API and suitable permissions. It does not prove the image can run, references exist, or storage will bind. `diff` normally returns exit code 1 when changes are present. `apply` is the step that changes cluster state.

See the command references for [apply and dry-run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/) and [diff exit codes](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_diff/).

Create dependencies intentionally: namespace first; configuration/storage before workloads that require them. `kubectl apply -f DIRECTORY/` reads supported manifest files directly in a directory; use `-R` for recursive directories. Do not rely on filename sorting as a dependency mechanism. A plain manifest directory is different from a Kustomize directory, which uses `kubectl apply -k DIRECTORY/`.

## Final writing checklist

- [ ] Right context, API version, kind, name, and namespace?
- [ ] Labels on the Pods, and selectors matching those labels?
- [ ] Pod fields and container fields at the correct nesting level?
- [ ] Application actually listens on the Service's target port?
- [ ] Secret/ConfigMap keys and PVC/volume names connected correctly?
- [ ] Strings quoted where required; list items and indentation correct?
- [ ] Generated status/bookkeeping removed and real credentials excluded?
- [ ] Server dry-run/diff reviewed, then runtime readiness checked after apply?

**Memory cue: identify → nest → connect → validate.**
