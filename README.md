# iot-streaming-platform-manifests

Deployment state for [iot-streaming-platform](https://github.com/heeyeon-yang/iot-streaming-platform), split into its own repo so ArgoCD has a single source of truth to watch and the app repo's commit history isn't mixed with deploy noise.

## This repo is not edited by hand

Everything under `base/` is Kustomize manifests, and image tags here are updated automatically by the app repo's CI (`build-and-deploy.yml`) after each image build. The `github-actions[bot]` commits ("chore: update image tags to ...") are that automation running, not manual work. If you're looking for application code or Terraform, that's all in the app repo.

## Layout

- `argocd/application.yaml` — the ArgoCD Application. Points at this repo's `base/` path, targets the `iot-streaming` namespace, syncs with `automated + prune + selfHeal` so cluster state can't drift from what's committed here.
- `base/` — namespace, service accounts, services, configmap, and deployments for the four services (ingestion-api, read-api, stream-processor, alerting-service).

## Secrets aren't committed here

`base/secret-alerting.example.yaml` shows the shape of the `alerting-secrets` Secret (just `SLACK_WEBHOOK_URL`), but the real one is created directly in-cluster with `kubectl create secret` and is gitignored. An earlier version of this had the actual webhook URL committed (base64, but still plaintext-recoverable) in a public repo. The webhook was revoked and reissued, and the secret was pulled out of Kustomize entirely so ArgoCD's `selfHeal` can't reintroduce it from git.

## How a deploy happens

A push to `main` in the app repo that touches `services/**` triggers GitHub Actions, which builds and pushes the four images to ECR, then clones this repo and runs `kustomize edit set image` for each service before committing the new tags. ArgoCD picks up that commit and reconciles the cluster.
