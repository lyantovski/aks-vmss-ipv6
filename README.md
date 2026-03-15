# aks-vmss-ipv6

Per-node IPv6 egress and ingress for AKS dual-stack clusters using an Azure Standard Load Balancer, Public IPv6 Prefixes, and VMSS autoscaler — fully automated with a Kubernetes DaemonSet and CronJob.

## Architecture

![Architecture Diagram](./docs/architecture.png)

Each AKS node gets a **dedicated 1:1 mapping** on an Azure Standard Load Balancer:

| Layer | Resource | Purpose |
|---|---|---|
| Frontend IP | `<computerName>-pip6` | Public IPv6 from a `/124` prefix |
| Backend Pool | `<computerName>-BE-ipv6` | Node's private IPv6 address |
| Outbound Rule | `<computerName>-out-rule` | IPv6 egress (all protocols) |
| Inbound NAT Rule | `<computerName>-in-rule` | IPv6 ingress (TCP 8080 → node 8080) |

**Key design points:**

- **Outbound-type `none`** — the AKS cluster has no default internet egress; IPv4 flows via private endpoints (ACR, API server). IPv6 egress is provided exclusively through this LB.
- **Private cluster** — API server is accessed via private endpoint from a jump VM.
- **Overlay CNI (dual-stack)** — pods get both IPv4 (`10.244.x.x`) and IPv6 (`fd12:3456:789a::x`) addresses; nodes sit on VNet with `172.16.1.0/24` + `fdf0:1e2d:3c4b::/64`.
- **IPv6 prefix rotation** — PIPs are allocated from `/124` prefixes (16 IPs each). When full, the automation creates the next prefix automatically.
- **Autoscaler-safe** — the DaemonSet runs on every node, so new VMSS instances are configured automatically. The CronJob cleans up orphaned resources when nodes scale down.

## Repository layout

| Folder | Description |
|---|---|
| [`ipv6-vmss-manager/`](ipv6-vmss-manager/) | **Automated solution** — DaemonSet, CronJob, ConfigMap, ServiceAccount, and docs |
| [`manual-steps-ipv6-vmss/`](manual-steps-ipv6-vmss/) | Manual step-by-step commands used during initial development and testing |
| [`docs/`](docs/) | Architecture diagram and supporting assets |

## Quick start

See the [detailed guide](ipv6-vmss-manager/readme.md) for the full walkthrough including infrastructure setup.

For the automation deployment only:

```bash
# 1. Workload Identity prerequisites (one-time)
#    See ipv6-vmss-manager/readme.md for setup steps

# 2. Deploy the automation
kubectl apply -f ipv6-vmss-manager/service-account.yaml
kubectl apply -f ipv6-vmss-manager/configmap.yaml
kubectl apply -f ipv6-vmss-manager/daemonset.yaml
kubectl apply -f ipv6-vmss-manager/cronjob.yaml
```

## Documentation

| Document | Audience |
|---|---|
| [ipv6-vmss-manager/readme.md](ipv6-vmss-manager/readme.md) | Full guide — infra setup, manual steps, automation, troubleshooting |