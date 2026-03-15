# 1. Variables - Update these
RESOURCE_GROUP="island-cilium-rg"
CLUSTER_NAME="island-cilium-aks"
LOCATION="swedencentral"
IDENTITY_NAME="ipv6-manager-identity"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# 2. Enable Workload Identity on AKS
az aks update -g $RESOURCE_GROUP -n $CLUSTER_NAME \
    --enable-oidc-issuer \
    --enable-workload-identity

# 3. Create Managed Identity
az identity create --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
USER_ASSIGNED_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv)

USER_ASSIGNED_PRINCIPAL_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query principalId -o tsv)

Output:
$ az identity create --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
{
  "clientId": "e905e62c-e90d-4ba0-a0bd-113208bc385a",
  "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourcegroups/island-cilium-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ipv6-manager-identity",
  "location": "swedencentral",
  "name": "ipv6-manager-identity",
  "principalId": "4a4802f7-3e4d-4360-a9bd-f79296a17b74",
  "resourceGroup": "island-cilium-rg",
  "systemData": null,
  "tags": {},
  "tenantId": "b393e737-e60a-43b1-828e-04d7edee2de1",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}
$ echo $USER_ASSIGNED_CLIENT_ID
e905e62c-e90d-4ba0-a0bd-113208bc385a

$ echo $USER_ASSIGNED_PRINCIPAL_ID
4a4802f7-3e4d-4360-a9bd-f79296a17b74

# 4. Grant Network Permissions (Scope to the Node Resource Group)
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query nodeResourceGroup -o tsv)
az role assignment create --role "Network Contributor" \
    --assignee $USER_ASSIGNED_CLIENT_ID \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RG"

az role assignment create --role "Reader" \
    --assignee $USER_ASSIGNED_CLIENT_ID \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RG"

az role assignment create --role "Network Contributor" \
    --assignee-object-id $USER_ASSIGNED_PRINCIPAL_ID \
    --assignee-principal-type "ServicePrincipal" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RG"

az role assignment create \
    --assignee-object-id $USER_ASSIGNED_PRINCIPAL_ID \
    --assignee-principal-type "ServicePrincipal" \
    --role "Network Contributor" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

Output:
$ echo $NODE_RG
MC_island-cilium-rg_island-cilium-aks_swedencentral


# 5. Get OIDC Issuer URL
AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER_NAME -g $RESOURCE_GROUP --query "oidcIssuerProfile.issuerUrl" -o tsv)"

Output:
$ echo $AKS_OIDC_ISSUER
https://swedencentral.oic.prod-aks.azure.com/b393e737-e60a-43b1-828e-04d7edee2de1/35232d3a-03ec-417d-be11-fe79d0bc7242/

# 6. Create Federated Credentials for the ServiceAccount
# We bind the identity to the ServiceAccount in kube-system
az identity federated-credential create --name "ipv6-manager-fed" \
    --identity-name $IDENTITY_NAME --resource-group $RESOURCE_GROUP \
    --issuer "${AKS_OIDC_ISSUER}" \
    --subject "system:serviceaccount:kube-system:ipv6-manager-sa" \
    --audience api://AzureADTokenExchange

Output:
$ az identity federated-credential create --name "ipv6-manager-fed" \
    --identity-name $IDENTITY_NAME --resource-group $RESOURCE_GROUP \
    --issuer "${AKS_OIDC_ISSUER}" \
    --subject "system:serviceaccount:kube-system:ipv6-manager-sa" \
    --audience api://AzureADTokenExchange
{
  "audiences": [
    "api://AzureADTokenExchange"
  ],
  "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourcegroups/island-cilium-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ipv6-manager-identity/federatedIdentityCredentials/ipv6-manager-fed",
  "issuer": "https://swedencentral.oic.prod-aks.azure.com/b393e737-e60a-43b1-828e-04d7edee2de1/35232d3a-03ec-417d-be11-fe79d0bc7242/",
  "name": "ipv6-manager-fed",
  "resourceGroup": "island-cilium-rg",
  "subject": "system:serviceaccount:kube-system:ipv6-manager-sa",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
}



After scale to 17 pods:
$ kubectl get nodes -o wide
NAME                             STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP              OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-default-47vwq                Ready    <none>   16m     v1.33.7   172.16.1.19   2603:1020:1001:23::3ff   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-4czpm                Ready    <none>   17m     v1.33.7   172.16.1.7    2603:1020:1001:23::3f3   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-4d6gb                Ready    <none>   16m     v1.33.7   172.16.1.14   2603:1020:1001:23::3f9   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-dnbhg                Ready    <none>   18m     v1.33.7   172.16.1.5    2603:1020:1001:23::3f2   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-fq724                Ready    <none>   16m     v1.33.7   172.16.1.18   2603:1020:1001:23::3fd   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-gvwtp                Ready    <none>   17m     v1.33.7   172.16.1.10   2603:1020:1001:23::3f1   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-hvvft                Ready    <none>   4h10m   v1.33.7   172.16.1.6    2603:1020:1001:23::3f0   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-j5dtx                Ready    <none>   17m     v1.33.7   172.16.1.12   2603:1020:1001:23::3f6   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-l7xxj                Ready    <none>   17m     v1.33.7   172.16.1.13   2603:1020:1001:23::3f8   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-lr7ws                Ready    <none>   17m     v1.33.7   172.16.1.8    2603:1020:1001:23::3f4   Ubuntu 22.04.5 LTS   5.15.0-1102-azutainerd://1.7.30-2
aks-default-pvtf8                Ready    <none>   10m     v1.33.7   172.16.1.21   2603:1020:1001:22::410   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-qlm9l                Ready    <none>   16m     v1.33.7   172.16.1.15   2603:1020:1001:23::3fb   Ubuntu 22.04.5 LTS   5.15.0-1102-azum     v1.33.7   172.16.1.20   2603:1020:1001:23::3fe   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-zh7tm                Ready    <none>   16m     v1.33.7   172.16.1.17   2603:1020:1001:23::3fc   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-40454228-vmss000000   Ready    <none>   6h36m   v1.33.7   172.16.1.4    <none>                   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2


Log of Daemonset:
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "b393e737-e60a-43b1-828e-04d7edee2de1",
    "id": "a303e0dc-f916-4c4a-8af1-e40f6163e1bb",
    "isDefault": true,
    "managedByTenants": [
      {
        "tenantId": "72f988bf-86f1-41af-91ab-2d7cd011db47"
      }
    ],
    "name": "ME-MngEnvMCAP948292-lyantovski-1",
    "state": "Enabled",
    "tenantId": "b393e737-e60a-43b1-828e-04d7edee2de1",
    "user": {
      "name": "e905e62c-e90d-4ba0-a0bd-113208bc385a",
      "type": "servicePrincipal"
    }
  }
]
Attempting to create IP in prefix: aks-ipv6-prefix-1...
Prefix aks-ipv6-prefix-1 is actually full. Trying next prefix...
Prefix aks-ipv6-prefix-2 does not exist. Creating new prefix...
Attempting to create IP in prefix: aks-ipv6-prefix-2...
Successfully created Public IP in aks-ipv6-prefix-2.
Attaching ipconfig-v6 to NIC: aks-default-pvtf8 using Subnet: /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks
{
  "etag": "W/\"6356bc5d-7ca0-46db-985d-48c7de6529b6\"",
  "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/networkInterfaces/aks-default-pvtf8/ipConfigurations/ipconfig-v6",
  "name": "ipconfig-v6",
  "privateIPAddress": "fdf0:1e2d:3c4b::14",
  "privateIPAddressVersion": "IPv6",
  "privateIPAllocationMethod": "Dynamic",
  "provisioningState": "Succeeded",
  "publicIPAddress": {
    "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-pvtf8",
    "resourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral"
  },
  "resourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral",
  "subnet": {
    "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks",
    "resourceGroup": "island-cilium-rg"
  },
  "type": "Microsoft.Network/networkInterfaces/ipConfigurations"
}

Log of Cronjob:

[Sat Mar  7 20:45:30 UTC 2026] Authenticating to Azure via Workload Identity...
[Sat Mar  7 20:45:33 UTC 2026] Starting scan in Resource Group: MC_island-cilium-rg_island-cilium-aks_swedencentral
[Sat Mar  7 20:45:34 UTC 2026] Found 1 orphaned resources to clean up.
------------------------------------------------------------
[Sat Mar  7 20:45:34 UTC 2026] RESOURCE FOUND:
  NAME: pip-v6-aks-default-pvtf9
  IP:   2603:1020:1001:22::411
  ID:   /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-pvtf9
[Sat Mar  7 20:45:34 UTC 2026] Action: Attempting deletion of pip-v6-aks-default-pvtf9...
[Sat Mar  7 20:45:36 UTC 2026] Result: Successfully deleted pip-v6-aks-default-pvtf9.
------------------------------------------------------------
[Sat Mar  7 20:45:36 UTC 2026] Cleanup job finished.

Log of Cronjob - delete 16 orphan ip addresses:
[Sat Mar  7 20:55:46 UTC 2026] Result: Successfully deleted pip-v6-aks-default-qlm9l.
------------------------------------------------------------
[Sat Mar  7 20:55:46 UTC 2026] RESOURCE FOUND:
  NAME: pip-v6-aks-default-rhhqz
  IP:   2603:1020:1001:23::3fa
  ID:   /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-rhhqz
[Sat Mar  7 20:55:46 UTC 2026] Action: Attempting deletion of pip-v6-aks-default-rhhqz...
[Sat Mar  7 20:55:58 UTC 2026] Result: Successfully deleted pip-v6-aks-default-rhhqz.
------------------------------------------------------------
[Sat Mar  7 20:55:58 UTC 2026] RESOURCE FOUND:
  NAME: pip-v6-aks-default-schrx
  IP:   2603:1020:1001:23::3f5
  ID:   /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-schrx
[Sat Mar  7 20:55:58 UTC 2026] Action: Attempting deletion of pip-v6-aks-default-schrx...
[Sat Mar  7 20:56:00 UTC 2026] Result: Successfully deleted pip-v6-aks-default-schrx.
------------------------------------------------------------
[Sat Mar  7 20:56:00 UTC 2026] RESOURCE FOUND:
  NAME: pip-v6-aks-default-stzcz
  IP:   2603:1020:1001:23::3f7
  ID:   /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-stzcz
[Sat Mar  7 20:56:00 UTC 2026] Action: Attempting deletion of pip-v6-aks-default-stzcz...
[Sat Mar  7 20:56:02 UTC 2026] Result: Successfully deleted pip-v6-aks-default-stzcz.
------------------------------------------------------------
[Sat Mar  7 20:56:02 UTC 2026] RESOURCE FOUND:
  NAME: pip-v6-aks-default-xmzdp
  IP:   2603:1020:1001:23::3fe
  ID:   /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-xmzdp
[Sat Mar  7 20:56:02 UTC 2026] Action: Attempting deletion of pip-v6-aks-default-xmzdp...
[Sat Mar  7 20:56:14 UTC 2026] Result: Successfully deleted pip-v6-aks-default-xmzdp.
------------------------------------------------------------
[Sat Mar  7 20:56:14 UTC 2026] RESOURCE FOUND:
  NAME: pip-v6-aks-default-zh7tm
  IP:   2603:1020:1001:23::3fc
  ID:   /subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/pip-v6-aks-default-zh7tm
[Sat Mar  7 20:56:14 UTC 2026] Action: Attempting deletion of pip-v6-aks-default-zh7tm...
[Sat Mar  7 20:56:16 UTC 2026] Result: Successfully deleted pip-v6-aks-default-zh7tm.
------------------------------------------------------------
[Sat Mar  7 20:56:16 UTC 2026] Cleanup job finished.

When no IPv6 addresses were found to delete:
[Sat Mar  7 21:00:01 UTC 2026] Authenticating to Azure via Workload Identity...
[Sat Mar  7 21:00:05 UTC 2026] Starting scan in Resource Group: MC_island-cilium-rg_island-cilium-aks_swedencentral
[Sat Mar  7 21:00:06 UTC 2026] Scan complete: No orphaned IPv6 addresses found.
