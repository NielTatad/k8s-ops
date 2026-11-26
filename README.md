# k8s-ops

This repository manages Kubernetes operational resources for the local Floci
EKS environment. Kubernetes YAML is the source of truth, and Kustomize composes
the resources for each target cluster. The current scope is intentionally small:
the `monitoring` and `infrastructure` namespaces only.

## Layout

```text
.
|-- clusters/
|   `-- floci-eks/             # Cluster overlay
|-- namespaces/
|   |-- infrastructure/        # infrastructure Namespace package
|   `-- monitoring/            # monitoring Namespace package
`-- .github/workflows/         # CI validation and manual CD deployment
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
