# ipv6-vmss-manager

Automatically configures an Azure Standard Load Balancer with per-node IPv6 public addresses for AKS VMSS nodes. Runs as a DaemonSet so every node in the cluster gets its own LB setup.

## What it does (per node)

1. **Backend pool** — creates an LB backend address pool using the node's private IPv6 address.
2. **Public IPv6 from prefix** — allocates a public IPv6 address from an IPv6 prefix (`/124`). If the current prefix is full, it creates a new one and retries.
3. **Frontend IP** — assigns the public IPv6 as a frontend IP on the load balancer.
4. **Outbound rule** — creates an outbound rule (all protocols) linking the frontend to the backend pool.
5. **Inbound NAT rule** — creates a TCP inbound NAT rule on port 8080 (configurable via `NAT_PORT`).

All steps are **idempotent** — re-running the script on the same node is safe.

## Prerequisites

- AKS cluster with dual-stack (IPv4 + IPv6) networking.
- Workload Identity enabled on the cluster (see setup below).

## Workload Identity Setup

### 1. Enable Workload Identity on the AKS cluster

If not already enabled during cluster creation, update the existing cluster:

```bash
az aks update \
  -g <RESOURCE_GROUP> \
  -n <CLUSTER_NAME> \
  --enable-oidc-issuer \
  --enable-workload-identity
```

### 2. Get the OIDC issuer URL

```bash
OIDC_ISSUER=$(az aks show \
  -g <RESOURCE_GROUP> \
  -n <CLUSTER_NAME> \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)
echo "$OIDC_ISSUER"
```

### 3. Create a User-Assigned Managed Identity

```bash
IDENTITY_NAME="ipv6-vmss-manager-identity"
RG="<RESOURCE_GROUP>"
MC_RG="<NODE_RESOURCE_GROUP>"   # e.g. MC_island-rg_island-aks-byo_swedencentral

az identity create --name "$IDENTITY_NAME" --resource-group "$RG"

IDENTITY_CLIENT_ID=$(az identity show -n "$IDENTITY_NAME" -g "$RG" --query clientId -o tsv)
IDENTITY_PRINCIPAL_ID=$(az identity show -n "$IDENTITY_NAME" -g "$RG" --query principalId -o tsv)
echo "Client ID:    $IDENTITY_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"
```

### 4. Assign RBAC roles on the node resource group

The identity needs permissions to manage networking resources and read VM/NIC metadata in the `MC_*` resource group:

```bash
# Network Contributor — manage LB, PIPs, prefixes
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Network Contributor" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/$MC_RG"

# Virtual Machine Contributor — read VM and NIC metadata
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Virtual Machine Contributor" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/$MC_RG"
```

### 5. Create a Federated Credential

This links the Kubernetes ServiceAccount to the managed identity:

```bash
az identity federated-credential create \
  --name "ipv6-vmss-manager-fc" \
  --identity-name "$IDENTITY_NAME" \
  --resource-group "$RG" \
  --issuer "$OIDC_ISSUER" \
  --subject "system:serviceaccount:kube-system:ipv6-vmss-manager-sa" \
  --audiences "api://AzureADTokenExchange"
```

### 6. Update the ServiceAccount with the Client ID

Replace the placeholder in [service-account.yaml](service-account.yaml):

```yaml
azure.workload.identity/client-id: "<paste $IDENTITY_CLIENT_ID here>"
```

Or with sed:

```bash
sed -i "s/<YOUR_MANAGED_IDENTITY_CLIENT_ID>/$IDENTITY_CLIENT_ID/" ipv6-vmss-manager/service-account.yaml
```

## Configuration

Edit the data section in [configmap.yaml](configmap.yaml):

| Variable | Default | Description |
|---|---|---|
| `LB_NAME` | `island-ext-lb` | Name of the Standard Load Balancer |
| `VNET_NAME` | `island-vnet` | VNet containing the AKS subnet |
| `PREFIX_BASE_NAME` | `aks-lb-ipv6-prefix` | Base name for IPv6 public IP prefixes |
| `NAT_PORT` | `8080` | Port for the inbound NAT rule |

## Deployment

```bash
kubectl apply -f ipv6-vmss-manager/service-account.yaml
kubectl apply -f ipv6-vmss-manager/configmap.yaml
kubectl apply -f ipv6-vmss-manager/daemonset.yaml
```

## Files

| File | Purpose |
|---|---|
| `service-account.yaml` | ServiceAccount with workload identity annotation |
| `configmap.yaml` | Environment variables and shell script for the LB setup |
| `daemonset.yaml` | DaemonSet that runs the script on every node |
