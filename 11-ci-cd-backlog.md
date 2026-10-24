# 11. CI/CD platform backlog

[Index](README.md) · Previous: [YAML writing manual](10-kubernetes-yaml-manual.md)

A service-independent, tickable path from a source commit to a running
Kubernetes workload. GitLab Self-Managed is the concrete CI/CD implementation;
the same delivery contracts can be carried to another CI system later. Start
with a tiny HTTP demo service, then apply the path to a real application.

## The delivery picture

```text
Developer → GitLab (source + pipeline + registry) → external Runner
                                                  ↓ build/test/push
                                         immutable image digest
                                                  ↓
GitLab deploy job → GitLab Agent → Minikube namespace → Helm release
                                                    → Service → client
```

GitLab and the runner live outside the workload cluster. The GitLab Agent
lives inside it with narrow deployment permissions. Minikube is the first
target environment; this single-node lab teaches delivery and recovery.

## How to use this backlog

- Follow the milestones and explicit `Depends on` links. Tick an acceptance
  check only after observing it, then tick the story and add a commit, merge
  request, pipeline, or runbook link.
- Keep project-specific build and test commands behind repository-owned
  scripts such as `ci/verify.sh`. The shared pipeline should accept inputs
  without knowing whether a service uses Java, Node, or another language.
- Split large stories into reviewable changes while preserving their ID and
  acceptance criteria as the contract.

## Shared definition of done

- A second person can reproduce the result from a clean checkout and
  documented external prerequisites.
- A failure path was exercised, is visible in the pipeline or cluster, and
  has a documented recovery action.
- Credentials come from external stores, are scoped to the job/environment,
  and are absent from source files and logs.
- Evidence ties the running result to the source commit and image digest.

## Milestone 1 — establish the delivery system

**Outcome:** A push starts a job on a self-managed runner, and a private image
can move from GitLab's registry into Minikube.

- [ ] **FOUND-01 — Define the delivery contract.** As a maintainer, I know
      which changes enter the pipeline and where they can deploy. Depends on:
      none.
  <details><summary>Implementation hint</summary>

  Record the GitLab host, runner host, registry DNS/TLS name, Minikube context, namespaces, branch policy, deployment owner, and one demo-service smoke URL in an architecture decision record. Draw the trust boundaries between source, runner, registry, agent, and workload namespace.
  </details>

  - [ ] A diagram identifies which component holds source, images, credentials,
        and cluster permissions.
  - [ ] The first environment, deploy trigger, smoke test, and rollback owner
        are written down before jobs are configured.

- [ ] **GITLAB-01 — Host source control and pipelines.** As a developer, I can
      push to a private project through a stable HTTPS/SSH address. Depends on:
      FOUND-01.
  <details><summary>Implementation hint</summary>

  Install GitLab Self-Managed on a supported host or VM outside Minikube. Configure a stable `external_url`, TLS, SSH, and a private project; pin the installed release and document its config, storage, and resource needs. Store administrator credentials and TLS keys outside the repo.
  </details>

  - [ ] An isolated client can clone, push, and open the project by its
        documented address.
  - [ ] Restarting GitLab preserves repositories and settings; stopping the
        workload namespace does not stop GitLab.

- [ ] **GITLAB-02 — Run jobs on a separate runner.** As a developer, I see a
      pipeline result for a test commit. Depends on: GITLAB-01.
  <details><summary>Implementation hint</summary>

  Install GitLab Runner outside Minikube; register a project-scoped runner with an authentication token. Choose and document an isolated executor (for example, Docker executor on a dedicated runner VM), explicit tags, and concurrency limits. Start with a harmless `.gitlab-ci.yml` job printing tool versions and commit SHA.
  </details>

  - [ ] A push queues and completes a tagged job on the intended runner.
  - [ ] A wrong tag leaves a job visibly pending; a stopped runner does not
        silently turn the pipeline green.

- [ ] **GITLAB-03 — Exchange images through the registry.** As a deployer, I
      can pull a private CI-built image into Minikube. Depends on: GITLAB-01,
      GITLAB-02.
  <details><summary>Implementation hint</summary>

  Enable the GitLab container registry with a trusted TLS endpoint. Push from CI using job-scoped registry credentials. Create a read-only deploy token for a Kubernetes `imagePullSecret`; verify that host, runner, and Minikube resolve and trust the same registry name.
  </details>

  - [ ] A job pushes a test image and records its digest; a new Pod pulls it
        with the read-only credential.
  - [ ] Revoking the pull credential causes the next pull to fail; no token or
        password is committed or printed in job logs.

## Milestone 2 — verify and publish a revision

**Outcome:** Merge requests are checked, and a passing default-branch commit
produces one traceable, immutable image.

- [ ] **CI-01 — Verify each change.** As a reviewer, I see build and test
      outcomes before merging. Depends on: GITLAB-02.
  <details><summary>Implementation hint</summary>

  Add `.gitlab-ci.yml` with `workflow: rules` for merge requests and the default branch. Run formatting/static checks, unit tests, and disposable-dependency integration tests through repository-owned scripts. Publish JUnit XML and relevant artifacts; use `needs` only for real job dependencies.
  </details>

  - [ ] A clean merge request displays the checks and test results.
  - [ ] A deliberately failing test blocks publish/deploy jobs; integration
        tests leave no shared mutable state.

- [ ] **CI-02 — Publish an immutable image.** As an operator, I can identify
      the source commit behind a running container. Depends on: CI-01,
      GITLAB-03.
  <details><summary>Implementation hint</summary>

  Build from a pinned Dockerfile or equivalent container build, tag with `CI_COMMIT_SHA`, push after verification, and capture the registry digest as a pipeline artifact or dotenv report. Deploy by digest or immutable SHA tag. Record the base-image and build-tool versions; never deploy a mutable `latest` tag.
  </details>

  - [ ] Only a green default-branch pipeline publishes the release image.
  - [ ] The pipeline links commit, tag, and digest; repeating deployment uses
        the same digest without rebuilding.

- [ ] **CI-03 — Inspect the release image.** As a reviewer, I can inspect its
      dependencies and known findings. Depends on: CI-02.
  <details><summary>Implementation hint</summary>

  Generate a CycloneDX or SPDX SBOM from the final image with Syft and scan its pinned digest with Trivy. Save both reports as pipeline artifacts, set a documented severity gate, and record time-limited exceptions with owner and reason. Pin scanner versions and refresh vulnerability data in CI.
  </details>

  - [ ] SBOM and scan report identify the pushed digest.
  - [ ] A deliberately introduced policy violation blocks release according
        to the gate, and its result is visible from the pipeline.

## Milestone 3 — deploy and recover

**Outcome:** The same tested image reaches Minikube through a controlled job;
a smoke test proves it works and a bad release is recoverable.

- [ ] **CD-01 — Describe the workload as a release.** As a deployer, I can
      render and install the demo service without editing live objects.
      Depends on: FOUND-01.
  <details><summary>Implementation hint</summary>

  Create a Helm chart with Deployment, ClusterIP Service, ConfigMap/Secret references, resource requests/limits, and startup/readiness/liveness probes. Parameterize image digest, hostname, and environment config. Use `helm lint` and `helm template` before a manual `helm upgrade --install` rehearsal.
  </details>

  - [ ] Rendered resources show the intended image, probes, Service, and
        configuration references with no real secrets or machine IPs.
  - [ ] A clean namespace installs the chart; a second upgrade is idempotent
        and serves the smoke URL.

- [ ] **CD-02 — Give GitLab scoped cluster access.** As an operator, I can
      deploy from CI without an administrator kubeconfig. Depends on:
      GITLAB-02, CD-01.
  <details><summary>Implementation hint</summary>

  Install the GitLab Agent for Kubernetes in Minikube, authorize the project for CI access, and bind its deployment identity to a namespace-scoped Role/RoleBinding with only the resource kinds and verbs the Helm release needs. Keep agent tokens outside source control; test a denied action in another namespace.
  </details>

  - [ ] A job reaches the intended context/namespace through the agent and
        cannot deploy into another namespace.
  - [ ] An out-of-scope permission test fails visibly; RBAC rules and agent
        bootstrap steps are documented without tokens.

- [ ] **CD-03 — Deploy the default branch automatically.** As a developer, I
      can merge a passing change and see the running version update. Depends
      on: CI-02, CI-03, CD-01, CD-02.
  <details><summary>Implementation hint</summary>

  Add a GitLab Environment and default-branch deploy job with `resource_group` to serialize releases; enable GitLab's Prevent outdated deployment jobs setting so an older pipeline cannot replace a newer release. Pass the published digest to `helm upgrade --install`, wait for readiness, and use the pinned Helm version's rollback-on-failure behavior. Run an HTTP smoke test and explicitly restore the previous release if that post-upgrade check fails. Record commit, digest, chart version, environment URL, and result.
  </details>

  - [ ] A green default-branch pipeline deploys its exact tested digest and
        records the deployment; merge requests cannot deploy.
  - [ ] A failing readiness or smoke check fails the job and retains/restores
        the previous working application release.
  - [ ] Concurrent pipelines cannot let an older release replace a newer one.

- [ ] **CD-04 — Promote without rebuilding.** As a release owner, I can move
      the same digest to a second environment after review. Depends on: CD-03.
  <details><summary>Implementation hint</summary>

  Add a second GitLab Environment and namespace with separate Helm values and Secrets. Make promotion a manual job restricted by branch/role policy available in the installed GitLab edition. Carry forward the exact digest from CI-02 and run the same smoke contract.
  </details>

  - [ ] Promotion uses the original digest without rebuilding.
  - [ ] Each environment has separate configuration, credentials, and
        deployment history; an unauthorized branch/user cannot promote.

- [ ] **CD-05 — Rehearse rollback and data compatibility.** As an operator, I
      can recover from a bad application release with a clear data decision.
      Depends on: CD-03.
  <details><summary>Implementation hint</summary>

  Deploy a broken image in a disposable environment, inspect the failed rollout, and redeploy the previous digest or use Helm rollback. For database-backed services, rehearse an expand/migrate/contract schema change with a versioned migration tool; prove the prior app version can still run against the expanded schema. Keep database restore separate from app rollback.
  </details>

  - [ ] A failed rollout leaves the last good app version serving traffic and
        its digest is identifiable.
  - [ ] A data-changing drill documents forward repair versus database
        restore; no pipeline assumes Helm undoes schema changes.

## Milestone 4 — operate and reuse the platform

**Outcome:** Common failures are diagnosable and a second service can reuse
the delivery path without copying secrets or fragile job definitions.

- [ ] **OPS-01 — Scope and rotate delivery credentials.** As an operator, I
      can rotate credentials without editing source. Depends on: GITLAB-03,
      CD-02.
  <details><summary>Implementation hint</summary>

  Inventory GitLab CI variables, runner tokens, registry deploy tokens, agent tokens, and Kubernetes Secrets. Use masked/protected variables where supported, a read-only pull token, namespace RBAC, and separate environment credentials. Document rotation and verify old credentials stop working.
  </details>

  - [ ] Each credential has an owner, scope, storage location, and rotation
        procedure; no secret is committed or logged.
  - [ ] Rotating one credential preserves the pipeline and invalidates the old
        credential.

- [ ] **OPS-02 — Diagnose a delivery failure.** As a developer, I can
      distinguish queue, test, registry, and rollout problems. Depends on:
      CD-03.
  <details><summary>Implementation hint</summary>

  Link job logs, JUnit reports, image digest, GitLab Environment history, `kubectl describe` events, Pod logs, and Helm status in a troubleshooting runbook. Record queue, build/test, and deploy times from several runs and investigate the largest bottleneck.
  </details>

  - [ ] A failed rollout can be traced from commit to pipeline to image to Pod.
  - [ ] The runbook reproduces one failure in each of the queue, registry, and
        workload layers with its first useful diagnostic command.

- [ ] **OPS-03 — Restore the delivery control plane.** As an administrator, I
      can recover projects, images, and pipeline history after host loss.
      Depends on: GITLAB-01, GITLAB-03.
  <details><summary>Implementation hint</summary>

  Schedule GitLab Self-Managed backups and copy archives off the GitLab host. Back up GitLab configuration/secrets separately and include registry or external object storage. Restore into an isolated installation of the same GitLab release/type, reconnect a runner, and pull an old image by digest.
  </details>

  - [ ] A drill recovers a project, repository, pipeline artifact, and registry
        image with documented commands.
  - [ ] The required version, configuration, secrets, and external storage are
        recorded; the restored system runs a new pipeline.

- [ ] **OPS-04 — Onboard a second service.** As a maintainer, I can reuse the
      delivery path for another language or repository. Depends on: CI-01,
      CD-03.
  <details><summary>Implementation hint</summary>

  Extract common jobs into a versioned GitLab CI include/component or repository template. Keep build/test scripts, chart path, smoke URL, runner tag, and namespace as explicit inputs. Onboard a second tiny service and pin the template revision; test a template change in both consumers.
  </details>

  - [ ] A second service gets verify, publish, and deploy jobs from documented
        inputs, with no copied secret values.
  - [ ] Both services pin and upgrade the template independently; a breaking
        template change is caught before deployment.

## Final demonstration

Open a merge request with a failing test, fix it, merge, inspect the published
digest, watch the Minikube rollout and smoke test, promote the same digest to a
second namespace, rehearse a bad rollout, then restore GitLab into an isolated
host and pull the original digest.

## Reference points

- [GitLab Self-Managed installation](https://docs.gitlab.com/install/)
- [GitLab Runner](https://docs.gitlab.com/runner/)
- [GitLab CI/CD YAML](https://docs.gitlab.com/ci/yaml/)
- [GitLab container registry authentication](https://docs.gitlab.com/user/packages/container_registry/authenticate_with_container_registry/)
- [GitLab Agent for Kubernetes deployments](https://docs.gitlab.com/user/clusters/agent/getting_started_deployments/)
- [GitLab environments](https://docs.gitlab.com/ci/environments/)
- [GitLab deployment safety](https://docs.gitlab.com/ci/environments/deployment_safety/)
- [GitLab backup and restore](https://docs.gitlab.com/administration/backup_restore/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)
