# Repository Guide for OpenCode Agents

Single-node K3s homelab cluster on Raspberry Pi 5 (`linux/arm64`), persistent NVMe storage at `/mnt/nvme`, Cloudflare Tunnel -> Traefik ingress routing.

## Communication & Agent Rules

- **ALWAYS ANSWER CONCISELY**: Give direct, succinct answers on both initial responses and all follow-ups. Avoid filler, fluff, and unnecessary explanations.

## Architecture & Workload Conventions

- **Manifest layout**: `manifests/<namespace>/<workload>.yaml`
  - Each namespace folder typically contains workload manifests, `limits.yaml` (`LimitRange`), and optional Traefik `redirect.yaml`.
  - Persistent host-path storage resides in `/mnt/nvme/k8s/data/<namespace>/`.
- **Target architecture**: `linux/arm64`. Any container images or binaries must support arm64.

## Secrets & Pre-commit Rules (CRITICAL)

- **Never commit secrets**:
  - `kind: Secret` inside tracked manifests is blocked by pre-commit hooks (`block-kind-secret`).
  - Secret manifests must be named `*.secret.yaml` (git-ignored, verified by `block-secret-files`).
