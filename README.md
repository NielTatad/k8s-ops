# k8s-ops

This repository manages Kubernetes operational resources for the local Floci
EKS environment. Kubernetes YAML is the source of truth, and Kustomize composes
the resources for each target cluster. The current scope is intentionally small:
the `monitoring` and `infrastructure` namespaces only.

## Layout

```text
.
|-- clusters/                  # Cluster-specific composition overlays
|-- namespaces/                # Reusable Namespace packages
|-- apps/                      # Self-contained application packages
|-- workloads/                 # Shared workload resource categories
|-- networking/                # Shared networking resource categories
|-- configuration/             # Shared ConfigMaps and secret templates
|-- security/                  # RBAC and ServiceAccount resources
|-- storage/                   # PVCs and StorageClasses
|-- scaling/                   # HPAs and PodDisruptionBudgets
|-- policies/                  # ResourceQuotas and LimitRanges
|-- monitoring/                # Future monitoring platform resources
|-- infrastructure/            # Future shared infrastructure resources
`-- .github/workflows/         # CI validation and CD deployment workflow
```

Namespaces are versioned Kubernetes `Namespace` resources. Managing them as
YAML gives their names, labels, ownership, review history, and recreation a
clear, repeatable source of truth.

## Kustomize structure

Kustomize normally uses **bases** for reusable resources and **overlays** for
cluster-specific composition or customization. In this repository, each
directory below `namespaces/` is a reusable resource package, while
`clusters/floci-eks/` is the overlay that composes them for Floci EKS. Add a
new cluster overlay rather than duplicating shared namespace manifests.

## Resource organization

Keep a resource next to the thing that owns it whenever possible. For example,
an application's Deployment, Service, ConfigMap, and Secret template should
live together in `apps/<application-name>/`. This makes the package portable
and straightforward to render independently.

Use the remaining top-level directories for shared or platform-owned resources:

- `workloads/`: shared Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs.
- `networking/`: shared Services, Ingresses, and NetworkPolicies.
- `configuration/`: shared ConfigMaps and encrypted-secret manifests or safe
  templates; application-specific configuration stays with its application.
- `security/`: ServiceAccounts and RBAC resources. Keep Roles and RoleBindings
  namespace-scoped; use ClusterRoles only when cluster-wide access is required.
- `storage/`: shared PersistentVolumeClaims and platform-owned StorageClasses.
- `scaling/`: HorizontalPodAutoscalers and PodDisruptionBudgets.
- `policies/`: namespace ResourceQuotas and LimitRanges.
- `monitoring/` and `infrastructure/`: future platform components, added only
  when there is a real manifest to manage.

Directories holding only a README are intentional placeholders. They do not
have `kustomization.yaml` files and are not deployable until real resources are
added. This avoids accidental inclusion of unfinished configuration.

## Adding an application

Create `apps/<application-name>/` with that application's manifests and a
`kustomization.yaml` that lists them. Then add `<application-name>` to the
`resources` list in `apps/kustomization.yaml`. Render the application directly
with `kubectl kustomize apps/<application-name>` before deciding whether a
specific cluster overlay should reference it.

Kubernetes Secrets are only base64-encoded by default; they are not encrypted
merely because they use the `Secret` kind. Use a local ignored file or an
encrypted workflow such as SOPS before introducing actual credentials. Never
commit a real password, token, certificate, or private key.

The `clusters/kind-services/` overlay is intentionally empty but valid. It
reserves the composition point for resources explicitly intended for the Kind
experiment cluster; it has no effect on Floci EKS.

## CI: validation

The `Validate Kubernetes Manifests` workflow runs on every push and pull
request using a GitHub-hosted runner. It:

- lints repository YAML with `yamllint`;
- builds `clusters/floci-eks` with Kustomize; and
- validates the rendered manifest with a client-side `kubectl` dry run.

CI has no kubeconfig, does not contact Floci or a Kubernetes cluster, and
never deploys resources.

## CD: manual deployment

The same workflow also exposes a manual **Run workflow** action. Its deployment
job runs only for `workflow_dispatch`, only after the validation job succeeds,
and only on a runner labelled `self-hosted` and `linux`. Pushes and pull
requests cannot deploy.

A self-hosted runner is required because GitHub-hosted runners cannot reach a
local Floci Kubernetes API at `https://localhost:6500`. The deployment job uses
`$HOME/.kube/config` already present on that runner, checks that the active
context is exactly `floci-eks`, confirms Kubernetes API access, previews the
overlay, and applies only `clusters/floci-eks`.

`http://localhost:4566` is Floci's AWS-compatible endpoint for tools such as
AWS CLI and Terraform. It is not used by `kubectl`; Kubernetes deployments use
the Kubernetes API from the kubeconfig instead. No AWS credentials are required
by this workflow.

## Self-hosted runner setup

Install and register a Linux GitHub Actions runner using the current commands
shown under the repository's **Settings > Actions > Runners > New self-hosted
runner** page. Run the runner as the same user that owns the existing
`$HOME/.kube/config`, or securely provide that user's kubeconfig to the runner
account without changing its contents. Do not commit the kubeconfig, its
certificates, or private keys.

The registration flow is:

1. On the Linux machine that can reach Floci, open the runner setup page above
   and select Linux.
2. Download and extract the runner package using the versioned commands GitHub
   provides there.
3. Run the displayed `./config.sh --url ... --token ...` command, adding
   `--labels self-hosted,linux` if those labels are not already assigned.
   Registration tokens are short-lived; never place one in this repository.
4. Start the runner with `./run.sh`, or configure its supplied service script
   so it runs under the intended Linux user.

The registered runner needs the labels `self-hosted` and `linux` (the default
labels supplied by a Linux self-hosted runner), plus:

- `kubectl` installed and available on `PATH`;
- read access to `$HOME/.kube/config` with an active `floci-eks` context;
- network access to the Kubernetes API at `https://localhost:6500`; and
- access to the local Docker/Floci environment when Floci requires it.

Keep the Kubernetes API private; do not expose port 6500 publicly. If the
runner itself is in a container, `localhost` means that container, not the
Floci host. Configure the container's network or kubeconfig endpoint so the
runner can reach the Kubernetes API before triggering deployment.

## Deploy safely

Before any local apply, verify that the active context is `floci-eks`:

```bash
kubectl config get-contexts
kubectl config current-context

kubectl kustomize clusters/floci-eks

kubectl diff -k clusters/floci-eks

kubectl apply -k clusters/floci-eks

kubectl get namespaces
```

`kubectl kustomize` builds and prints the manifest. `kubectl diff` previews the
changes Kubernetes would make and does not modify the cluster. `kubectl apply`
actually modifies the active cluster, so run it only after confirming the
context.

To deploy from GitHub, open **Actions**, select **Validate Kubernetes
Manifests**, choose **Run workflow**, and review the job output. The deployment
stops before applying if the runner has no kubeconfig, `kubectl` is unavailable,
the context is not `floci-eks`, the API is unreachable, or the overlay is
missing. After a successful run, verify with:

```bash
kubectl get namespaces
kubectl get all -n monitoring
kubectl get all -n infrastructure
```

## Local validation

```bash
yamllint .
kubectl kustomize clusters/floci-eks > /tmp/rendered-manifests.yaml
kubectl apply --dry-run=client -f /tmp/rendered-manifests.yaml
```

To explore the new packages without changing either cluster:

```bash
# Render the deployed Floci namespace overlay.
kubectl kustomize clusters/floci-eks

# Render the current application collection (empty until you add an app).
kubectl kustomize apps

# Preview a cluster overlay; this contacts the active cluster but makes no changes.
kubectl diff -k clusters/floci-eks

# Apply only after verifying the active context is floci-eks.
kubectl apply -k clusters/floci-eks
```

`kubectl apply -f` applies already-rendered YAML files. `kubectl apply -k`
renders a Kustomize directory and applies the result. `kubectl kustomize` only
prints the rendered result, while `kubectl diff -k` compares that result with
the active cluster without changing it.
