# Service Status Dashboard GitOps lab

This repository defines the desired state of a Next.js dashboard running on Minikube. GitHub Actions builds a Docker image from the [separate application repository](https://github.com/NaveenGumaste/service-status-dashboard), publishes it to GitHub Container Registry (GHCR), and commits the selected image to Helm values. Argo CD reads `main` and reconciles the cluster automatically.

## Repository layout

| Path | Purpose |
|---|---|
| [Build and promote workflow](.github/workflows/build-dashboard-image.yml) | Checks out the app, runs lint and tests, checks types, and smoke-tests the container. Publishes an image tagged with the app commit SHA and promotes it through Git. |
| [Helm chart](charts/service-status-dashboard) | Defines the Deployment, ClusterIP Service, and ConfigMap. |
| [Helm values](charts/service-status-dashboard/values.yaml) | Selects the image, configures two replicas, and supplies release metadata and probe simulation settings. |

## Verify the lab

The lab uses the `gitops-lab` Minikube context. The Argo CD application watches this repository’s `main` branch and renders `charts/service-status-dashboard` into the `service-status-dashboard` namespace. Auto-sync, self-heal, and prune are enabled. Argo CD uses a read-only deploy key for Git, and the workload uses the namespace’s `ghcr-pull` Secret for the private registry.

Run these checks from the GitOps repository root.

```powershell
helm lint ./charts/service-status-dashboard --strict
kubectl --context gitops-lab -n argocd get application service-status-dashboard
kubectl --context gitops-lab -n service-status-dashboard get deployment,pods,service
kubectl --context gitops-lab -n service-status-dashboard port-forward svc/service-status-dashboard 3010:80
```

Open `http://localhost:3010` while the port-forward runs. The dashboard displays the active release ID and commit SHA. Kubernetes uses `/api/ready` to determine whether a pod can receive traffic and `/api/health` to determine when to restart its container.

## Demo and results

- A manual release published app commit `3263dad` and updated the chart in GitOps commit `b8780f5`. Both pods ran the published image digest. [Successful release run](https://github.com/NaveenGumaste/service-status-dashboard-gitops/actions/runs/44921237237)
- A repeat run recognized the app commit in chart history and skipped the build and promotion. [Successful skip run](https://github.com/NaveenGumaste/service-status-dashboard-gitops/actions/runs/44922473469)
- The lab demonstrated replica drift repair, an unready pod excluded from Service traffic, and a container restart after failed liveness checks. Reverting the test commits in Git restored the chart configuration.
- The workflow also defines an hourly check at minute 17 UTC (minute 47 IST). Verification of a scheduled run is pending.
