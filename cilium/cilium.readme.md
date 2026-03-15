# ====== setup ======
LOCATION="swedencentral"
RG="island-cilium-rg"
VNET="island-cilium-vnet"
SUBNET="island-cilium-snet-aks"
VNET_V4="172.16.0.0/16"
VNET_V6="fdf0:1e2d:3c4b::/48"   
SUBNET_V4="172.16.1.0/24"
SUBNET_V6="fdf0:1e2d:3c4b::/64"

# Create Resource Group
az group create --name $RG --location $LOCATION

# Create VNet with IPv4 and IPv6 prefixes
az network vnet create \
    --resource-group $RG \
    --name $VNET \
    --address-prefixes $VNET_V4 $VNET_V6

# Create a Dual-Stack Subnet
az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET \
    --name $SUBNET \
    --address-prefixes $SUBNET_V4 $SUBNET_V6

#subnet id
SUBNET_ID=$(az network vnet subnet show -g "$RG" --vnet-name "$VNET" -n "$SUBNET" --query id -o tsv)
echo "$SUBNET_ID"

Output:
$ echo "$SUBNET_ID"
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks

# create managed identity
IDENTITY_NAME="island-cilium-aks-mi"
az identity create -g "$RG" -n "$IDENTITY_NAME" -l "$LOCATION"

Output:
az identity create -g "$RG" -n "$IDENTITY_NAME" -l "$LOCATION"
{
  "clientId": "8d2d1bdc-0a7f-4a48-a7b4-88d8e1f0144c",
  "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourcegroups/island-cilium-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/island-cilium-aks-mi",
  "location": "swedencentral",
  "name": "island-cilium-aks-mi",
  "principalId": "2b1cdb0a-7d79-4f6f-ae96-6eb13eed51e4",
  "resourceGroup": "island-cilium-rg",
  "systemData": null,
  "tags": {},
  "tenantId": "b393e737-e60a-43b1-828e-04d7edee2de1",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}


SUBSCRIPTION_ID=$(az account show --query id -o tsv)
IDENTITY_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME"
IDENTITY_PRINCIPAL_ID=$(az identity show --resource-group $RG --name $IDENTITY_NAME --query principalId -o tsv)
 
echo $IDENTITY_ID
echo $IDENTITY_PRINCIPAL_ID
echo $SUBSCRIPTION_ID

Output:
$ echo $IDENTITY_ID
echo $IDENTITY_PRINCIPAL_ID
echo $SUBSCRIPTION_ID
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourcegroups/island-cilium-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/island-cilium-aks-mi
2b1cdb0a-7d79-4f6f-ae96-6eb13eed51e4
a303e0dc-f916-4c4a-8af1-e40f6163e1bb

az role assignment create --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET" \
--role "Network Contributor" \
--assignee $IDENTITY_PRINCIPAL_ID

#AKS Cluster Creation
CLUSTER_NAME="island-cilium-aks"
POD_CIDR_V4="10.244.0.0/16"
SVC_CIDR_V4="10.0.0.0/16"
DNS_IP_V4="10.0.0.10"

az aks create \
    --resource-group $RG \
    --name $CLUSTER_NAME \
    --location $LOCATION \
    --vnet-subnet-id "$SUBNET_ID" \
    --assign-identity "$IDENTITY_ID" \
    --network-plugin none \
    --enable-managed-identity \
    --ip-families ipv4 \
    --node-provisioning-mode Auto \
    --nodepool-name "system" \
    --pod-cidrs "$POD_CIDR_V4" \
    --service-cidrs "$SVC_CIDR_V4" \
    --dns-service-ip "$DNS_IP_V4" \
    --node-count 1 \
    --generate-ssh-keys

Output:
$ az aks create \
    --resource-group $RG \
    --name $CLUSTER_NAME \
    --location $LOCATION \
    --vnet-subnet-id "$SUBNET_ID" \
    --assign-identity "$IDENTITY_ID" \
    --network-plugin none \
    --enable-managed-identity \
    --ip-families ipv4 \
    --node-provisioning-mode Auto \
    --nodepool-name "system" \
    --pod-cidrs "$POD_CIDR_V4" \
    --service-cidrs "$SVC_CIDR_V4" \
    --dns-service-ip "$DNS_IP_V4" \
    --node-count 1 \
    --generate-ssh-keys
Argument '--node-provisioning-mode' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
The behavior of this command has been altered by the following extension: aks-preview
The new node pool will enable SSH access, recommended to use '--ssh-access disabled' option to disable SSH access for the node pool to make it more secure.
{
  "aadProfile": null,
  "addonProfiles": null,
  "agentPoolProfiles": [
    {
      "artifactStreamingProfile": null,
      "availabilityZones": null,
      "capacityReservationGroupId": null,
      "count": 1,
      "creationData": null,
      "currentOrchestratorVersion": "1.33.7",
      "eTag": "50fe027f-2058-49c0-94f7-d0b8eabe40de",
      "enableAutoScaling": false,
      "enableCustomCaTrust": false,
      "enableEncryptionAtHost": false,
      "enableFips": false,
      "enableNodePublicIp": false,
      "enableUltraSsd": false,
      "gatewayProfile": null,
      "gpuInstanceProfile": null,
      "gpuProfile": null,
      "hostGroupId": null,
      "kubeletConfig": null,
      "kubeletDiskType": "OS",
      "linuxOsConfig": null,
      "localDnsProfile": null,
      "maxCount": null,
      "maxPods": 250,
      "messageOfTheDay": null,
      "minCount": null,
      "mode": "System",
      "name": "system",
      "networkProfile": {
        "allowedHostPorts": null,
        "applicationSecurityGroups": null,
        "nodePublicIpTags": null
      },
      "nodeImageVersion": "AKSUbuntu-2204gen2containerd-202602.13.5",
      "nodeInitializationTaints": null,
      "nodeLabels": null,
      "nodePublicIpPrefixId": null,
      "nodeTaints": null,
      "orchestratorVersion": "1.33",
      "osDiskSizeGb": 150,
      "osDiskType": "Ephemeral",
      "osSku": "Ubuntu",
      "osType": "Linux",
      "podIpAllocationMode": null,
      "podSubnetId": null,
      "powerState": {
        "code": "Running"
      },
      "provisioningState": "Succeeded",
      "proximityPlacementGroupId": null,
      "scaleDownMode": "Delete",
      "scaleSetEvictionPolicy": null,
      "scaleSetPriority": null,
      "securityProfile": {
        "enableSecureBoot": false,
        "enableVtpm": false,
        "sshAccess": "LocalUser"
      },
      "spotMaxPrice": null,
      "status": null,
      "tags": null,
      "type": "VirtualMachineScaleSets",
      "upgradeSettings": {
        "drainTimeoutInMinutes": null,
        "maxBlockedNodes": null,
        "maxSurge": "10%",
        "maxUnavailable": "0",
        "minSurge": null,
        "nodeSoakDurationInMinutes": null,
        "undrainableNodeBehavior": null
      },
      "upgradeSettingsBlueGreen": {
        "batchSoakDurationInMinutes": null,
        "drainBatchSize": null,
        "drainTimeoutInMinutes": null,
        "finalSoakDurationInMinutes": null
      },
      "upgradeStrategy": "Rolling",
      "virtualMachineNodesStatus": null,
      "virtualMachinesProfile": null,
      "vmSize": "Standard_D4ds_v4",
      "vnetSubnetId": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks",
      "windowsProfile": null,
      "workloadRuntime": "OCIContainer"
    }
  ],
  "aiToolchainOperatorProfile": null,
  "apiServerAccessProfile": null,
  "autoScalerProfile": null,
  "autoUpgradeProfile": {
    "nodeOsUpgradeChannel": "NodeImage",
    "upgradeChannel": null
  },
  "azureMonitorProfile": null,
  "azurePortalFqdn": "island-cil-island-cilium-rg-a303e0-kl2vba56.portal.hcp.swedencentral.azmk8s.io",
  "bootstrapProfile": {
    "artifactSource": "Direct",
    "containerRegistryId": null
  },
  "creationData": null,
  "currentKubernetesVersion": "1.33.7",
  "disableLocalAccounts": false,
  "diskEncryptionSetId": null,
  "dnsPrefix": "island-cil-island-cilium-rg-a303e0",
  "eTag": "9ddeee53-fc0c-4ad6-aecb-b9cbb178596b",
  "enableNamespaceResources": null,
  "enableRbac": true,
  "extendedLocation": null,
  "fqdn": "island-cil-island-cilium-rg-a303e0-kl2vba56.hcp.swedencentral.azmk8s.io",
  "fqdnSubdomain": null,
  "httpProxyConfig": null,
  "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourcegroups/island-cilium-rg/providers/Microsoft.ContainerService/managedClusters/island-cilium-aks",
  "identity": {
    "delegatedResources": null,
    "principalId": null,
    "tenantId": null,
    "type": "UserAssigned",
    "userAssignedIdentities": {
      "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/island-cilium-aks-mi": {
        "clientId": "8d2d1bdc-0a7f-4a48-a7b4-88d8e1f0144c",
        "principalId": "2b1cdb0a-7d79-4f6f-ae96-6eb13eed51e4"
      }
    }
  },
  "identityProfile": {
    "kubeletidentity": {
      "clientId": "22359740-2b43-4eaf-900c-c969f88640db",
      "objectId": "5fa490fb-8c2d-4b7d-bbb1-1d0a734f10c6",
      "resourceId": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourcegroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.ManagedIdentity/userAssignedIdentities/island-cilium-aks-agentpool"
    }
  },
  "ingressProfile": null,
  "kind": "Base",
  "kubernetesVersion": "1.33",
  "linuxProfile": {
    "adminUsername": "azureuser",
    "ssh": {
      "publicKeys": [
        {
          "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCq8olBcW4dmLx4bGYI6dPdsRjJSktVlS/CzQ0+ZiIuO+/nHYUXBDdcX+UX4PW9IhNLR4oqDXYlT9Nph3cohCcZGSIKuVtrrfqWgPS0QpHoWStfrOO6JRth1uLzmlldBfnCFl6tq2snpUy2SBgn6fa1UQG/dmU39SX3YxJP3A+TGuekkZVMmTI8Mk1Cip7YUOylUSurJRqUBi0xNbSxj33h+Kaa7tvGFxG6O7t/Bd3H5+0zmjwWTz1x8dPwT1LIVB0rNxoiUnkdI0m10R6WgDZG7s60en/zZk+jw7+xbAkHGFAdo9cT7oNYr7aPciqn+KQX+SNrG8hh9YPpdzi4QVkV"
        }
      ]
    }
  },
  "location": "swedencentral",
  "maxAgentPools": 100,
  "metricsProfile": {
    "costAnalysis": {
      "enabled": false
    }
  },
  "name": "island-cilium-aks",
  "networkProfile": {
    "advancedNetworking": null,
    "dnsServiceIp": "10.0.0.10",
    "ipFamilies": [
      "ipv4"
    ],
    "kubeProxyConfig": null,
    "loadBalancerProfile": {
      "allocatedOutboundPorts": null,
      "backendPoolType": "nodeIPConfiguration",
      "clusterServiceLoadBalancerHealthProbeMode": null,
      "effectiveOutboundIPs": [
        {
          "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/54df2855-6e7c-4b81-a601-df78711b2bfd",
          "resourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral"
        }
      ],
      "enableMultipleStandardLoadBalancers": null,
      "idleTimeoutInMinutes": null,
      "managedOutboundIPs": {
        "count": 1,
        "countIpv6": null
      },
      "outboundIPs": null,
      "outboundIpPrefixes": null
    },
    "loadBalancerSku": "standard",
    "natGatewayProfile": null,
    "networkDataplane": null,
    "networkMode": null,
    "networkPlugin": "none",
    "networkPluginMode": null,
    "networkPolicy": "none",
    "outboundType": "loadBalancer",
    "podCidr": "10.244.0.0/16",
    "podCidrs": [
      "10.244.0.0/16"
    ],
    "podLinkLocalAccess": "IMDS",
    "serviceCidr": "10.0.0.0/16",
    "serviceCidrs": [
      "10.0.0.0/16"
    ],
    "staticEgressGatewayProfile": null
  },
  "nodeProvisioningProfile": {
    "defaultNodePools": "Auto",
    "mode": "Auto"
  },
  "nodeResourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral",
  "nodeResourceGroupProfile": null,
  "oidcIssuerProfile": {
    "enabled": false,
    "issuerUrl": null
  },
  "podIdentityProfile": null,
  "powerState": {
    "code": "Running"
  },
  "privateFqdn": null,
  "privateLinkResources": null,
  "provisioningState": "Succeeded",
  "publicNetworkAccess": null,
  "resourceGroup": "island-cilium-rg",
  "resourceUid": "69ac2e432dd1a0000196ffef",
  "schedulerProfile": null,
  "securityProfile": {
    "azureKeyVaultKms": null,
    "customCaTrustCertificates": null,
    "defender": null,
    "imageCleaner": null,
    "imageIntegrity": null,
    "kubernetesResourceObjectEncryptionProfile": null,
    "nodeRestriction": null,
    "workloadIdentity": null
  },
  "serviceMeshProfile": null,
  "servicePrincipalProfile": {
    "clientId": "msi",
    "secret": null
  },
  "sku": {
    "name": "Base",
    "tier": "Free"
  },
  "status": null,
  "storageProfile": {
    "blobCsiDriver": null,
    "diskCsiDriver": {
      "enabled": true,
      "version": "v1"
    },
    "fileCsiDriver": {
      "enabled": true
    },
    "snapshotController": {
      "enabled": true
    }
  },
  "supportPlan": "KubernetesOfficial",
  "systemData": null,
  "tags": null,
  "type": "Microsoft.ContainerService/ManagedClusters",
  "upgradeSettings": null,
  "windowsProfile": {
    "adminPassword": null,
    "adminUsername": "azureuser",
    "enableCsiProxy": true,
    "gmsaProfile": null,
    "licenseType": null
  },
  "workloadAutoScalerProfile": {
    "keda": null,
    "verticalPodAutoscaler": null
  }
}

az aks get-credentials --resource-group $RG --name $CLUSTER_NAME --overwrite-existing

$ kubectl get nodes
NAME                             STATUS     ROLES    AGE   VERSION
aks-system-40454228-vmss000000   NotReady   <none>   29m   v1.33.7
aks-system-surge-bjb8l           NotReady   <none>   25m   v1.33.7
aks-system-surge-gxcl9           NotReady   <none>   25m   v1.33.7
aks-system-surge-w4h86           NotReady   <none>   25m   v1.33.7

az aks nodepool update \
  -g "$RG" \
  -n "system" \
  --cluster-name "$CLUSTER_NAME" \
  --node-taints CriticalAddonsOnly=true:NoSchedule

https://github.com/cilium/cilium-cli/releases/download/v0.19.2/cilium-windows-amd64.zip
https://github.com/cilium/cilium/blob/main/install/kubernetes/cilium/README.md
https://github.com/cilium/cilium/tree/main/pkg/k8s/apis/cilium.io/client/crds/v2

#Install Cilium
$ helm repo add cilium https://helm.cilium.io/
$ helm repo update

API_SERVER_IP=$(kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}')
echo "API Server IP is: $API_SERVER_IP"

Output:
$ echo $API_SERVER_IP
9.223.13.112

helm install cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system \
  --set ipv4.enabled=true \
  --set ipv6.enabled=true \
  --set enableIPv6Masquerade=true \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=$API_SERVER_IP \
  --set k8sServicePort=443 \
  --set aksbyocni.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true \
  --set operator.replicas=2

$ kubectl get nodes -o wide
NAME                             STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-default-grbj5                Ready    <none>   5m50s   v1.33.7   172.16.1.8    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-40454228-vmss000000   Ready    <none>   53m     v1.33.7   172.16.1.4    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-surge-w4h86           Ready    <none>   49m     v1.33.7   172.16.1.7    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2


#Assign IPv6 public IP to node
VM="aks-default-grbj5"
MC_RG="MC_island-cilium-rg_island-cilium-aks_swedencentral"
PIP6_NAME="${VM}-pip6"
IP_CONFIG_NAME="ipconfig6"
 
NIC_ID=$(az vm show -g "$MC_RG" -n "$VM" --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC_NAME=$(basename "$NIC_ID")
echo $NIC_NAME

Output:
$ echo $NIC_ID
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/networkInterfaces/aks-default-grbj5
$ echo $NIC_NAME
aks-default-grbj5 

# assign ipv6 public ip to nic
 
az network public-ip create \
  -g "$MC_RG" -n "$PIP6_NAME" \
  --sku Standard \
  --version IPv6 \
  --allocation-method Static

Output:
$ az network public-ip create \
  -g "$MC_RG" -n "$PIP6_NAME" \
  --sku Standard \
  --version IPv6 \
  --allocation-method Static
[Coming breaking change] In the coming release, the default behavior will be changed as follows when sku is Standard and zone is not provided: For zonal regions, you will get a zone-redundant IP indicated by zones:["1","2","3"]; For non-zonal regions, you will get a non zone-redundant IP indicated by zones:null.
{
  "publicIp": {
    "ddosSettings": {
      "protectionMode": "VirtualNetworkInherited"
    },
    "etag": "W/\"4e7e1631-8259-4bc5-92c4-a5c5975716d8\"",
    "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/aks-default-grbj5-pip6",
    "idleTimeoutInMinutes": 4,
    "ipAddress": "2603:1020:1001:22::1b3",
    "ipTags": [],
    "location": "swedencentral",
    "name": "aks-default-grbj5-pip6",
    "provisioningState": "Succeeded",
    "publicIPAddressVersion": "IPv6",
    "publicIPAllocationMethod": "Static",
    "resourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral",
    "resourceGuid": "61803d6e-85b3-4d6f-94c3-141939ba1045",
    "sku": {
      "name": "Standard",
      "tier": "Regional"
    },
    "type": "Microsoft.Network/publicIPAddresses"
  }
}

So our IPv6 Public address: "2603:1020:1001:22::1b3"

# get subnet id
SUBNET_ID=$(az network nic show -g "$MC_RG" -n "$NIC_NAME" --query "ipConfigurations[0].subnet.id" -o tsv)
echo $SUBNET_ID

Output:
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks

az network nic ip-config create \
  -g "$MC_RG" --nic-name "$NIC_NAME" \
  -n "$IP_CONFIG_NAME" \
  --subnet "$SUBNET_ID" \
  --private-ip-address-version IPv6 \
  --public-ip-address "$PIP6_NAME"

Output:
$ az network nic ip-config create \
  -g "$MC_RG" --nic-name "$NIC_NAME" \
  -n "$IP_CONFIG_NAME" \
  --subnet "$SUBNET_ID" \
  --private-ip-address-version IPv6 \
  --public-ip-address "$PIP6_NAME"
{
  "etag": "W/\"23d7543e-33d5-4397-b810-b8161fa59c42\"",
  "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/networkInterfaces/aks-default-grbj5/ipConfigurations/ipconfig6",
  "name": "ipconfig6",
  "privateIPAddress": "fdf0:1e2d:3c4b::4",
  "privateIPAddressVersion": "IPv6",
  "privateIPAllocationMethod": "Dynamic",
  "provisioningState": "Succeeded",
  "publicIPAddress": {
    "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/publicIPAddresses/aks-default-grbj5-pip6",
    "resourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral"
  },
  "resourceGroup": "MC_island-cilium-rg_island-cilium-aks_swedencentral",
  "subnet": {
    "id": "/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks",
    "resourceGroup": "island-cilium-rg"
  },
  "type": "Microsoft.Network/networkInterfaces/ipConfigurations"
}

#Lets look on AKS Node aks-default-grbj5 (ipconfig6):
Public IP address: 2603:1020:1001:22::1b3
Private IP address: fdf0:1e2d:3c4b::4

helm upgrade cilium cilium/cilium -n kube-system \
  --reuse-values \
  --set k8sServiceHost=$API_SERVER_IP \
  --set k8sServicePort=443 \
  --set dnsproxy.enabled=true \
  --set kubeProxyReplacement=true

kubectl rollout restart deployment coredns -n kube-system

https://raw.githubusercontent.com/cilium/cilium/refs/heads/main/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumlocalredirectpolicies.yaml

helm upgrade cilium cilium/cilium -n kube-system \
  --reuse-values \
  --set localRedirectPolicies.enabled=true

apiVersion: "cilium.io/v2"
kind: CiliumLocalRedirectPolicy
metadata:
  name: "dns-hijack-global"
  namespace: kube-system
spec:
  redirectFrontend:
    # This matches the destination IP currently in your pods' /etc/resolv.conf
    addressMatcher:
      ip: "168.63.129.16"
      toPorts:
        - port: "53"
          protocol: UDP
          name: dns-udp
        - port: "53"
          protocol: TCP
          name: dns-tcp
  redirectBackend:
    # This tells Cilium to send that traffic to the CoreDNS pod on the SAME node
    localEndpointSelector:
      matchLabels:
        k8s-app: kube-dns
    toPorts:
      - port: "53"
        protocol: UDP
        name: dns-udp
      - port: "53"
        protocol: TCP
        name: dns-tcp

# application for tests
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hostnet
  namespace: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-hostnet
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx-hostnet
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
    # Force Anti-affinity
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: nginx-hostnet
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 8080
              hostPort: 8080
              protocol: TCP
          command: ["/bin/sh","-c"]
          args:
            - |
              cat >/etc/nginx/conf.d/default.conf <<'EOF'
              server {
                listen 8080;
                listen [::]:8080;
                server_name _;
                location / {
                  return 200 "nginx hostNetwork OK on $hostname\n";
                }
              }
              EOF
              nginx -g 'daemon off;'
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
--- 
apiVersion: v1
kind: Service
metadata:
  name: nginx-hostnet
  namespace: nginx-demo
  labels:
    app: nginx-hostnet
spec:
  selector:
    app: nginx-hostnet
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP 
 

Lets do IPv6 address assignment to new node: aks-default-q7ksk
#Assign IPv6 public IP to node
VM="aks-default-q7ksk"
MC_RG="MC_island-cilium-rg_island-cilium-aks_swedencentral"
PIP6_NAME="${VM}-pip6"
IP_CONFIG_NAME="ipconfig6"
 
NIC_ID=$(az vm show -g "$MC_RG" -n "$VM" --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC_NAME=$(basename "$NIC_ID")

Output:
$ echo $NIC_ID
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/MC_island-cilium-rg_island-cilium-aks_swedencentral/providers/Microsoft.Network/networkInterfaces/aks-default-q7ksk
$ echo $NIC_NAME
aks-default-q7ksk

# assign ipv6 public ip to nic
 
az network public-ip create \
  -g "$MC_RG" -n "$PIP6_NAME" \
  --sku Standard \
  --version IPv6 \
  --allocation-method Static

So our IPv6 Public address: "2603:1020:1001:4::13e"

# get subnet id
SUBNET_ID=$(az network nic show -g "$MC_RG" -n "$NIC_NAME" --query "ipConfigurations[0].subnet.id" -o tsv)
echo $SUBNET_ID

Output:
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks

az network nic ip-config create \
  -g "$MC_RG" --nic-name "$NIC_NAME" \
  -n "$IP_CONFIG_NAME" \
  --subnet "$SUBNET_ID" \
  --private-ip-address-version IPv6 \
  --public-ip-address "$PIP6_NAME"

Test connectivity: https://8gwifi.org/curlfunctions.jsp
curl -6 http://[2603:1020:1001:4::13e]:8080

/ # curl -6 http://[2603:1020:1001:4::13e]:8080
nginx hostNetwork OK on aks-default-q7ksk


##For node: aks-default-bsqxr
#Assign IPv6 public IP to node
VM="aks-default-bsqxr"
MC_RG="MC_island-cilium-rg_island-cilium-aks_swedencentral"
PIP6_NAME="${VM}-pip6"
IP_CONFIG_NAME="ipconfig6"
 
NIC_ID=$(az vm show -g "$MC_RG" -n "$VM" --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC_NAME=$(basename "$NIC_ID")

$ echo $NIC_NAME
aks-default-bsqxr

# assign ipv6 public ip to nic
 
az network public-ip create \
  -g "$MC_RG" -n "$PIP6_NAME" \
  --sku Standard \
  --version IPv6 \
  --allocation-method Static

So our IPv6 Public address: "2603:1020:1001:4::144"

# get subnet id
SUBNET_ID=$(az network nic show -g "$MC_RG" -n "$NIC_NAME" --query "ipConfigurations[0].subnet.id" -o tsv)
echo $SUBNET_ID

Output:
/subscriptions/a303e0dc-f916-4c4a-8af1-e40f6163e1bb/resourceGroups/island-cilium-rg/providers/Microsoft.Network/virtualNetworks/island-cilium-vnet/subnets/island-cilium-snet-aks

az network nic ip-config create \
  -g "$MC_RG" --nic-name "$NIC_NAME" \
  -n "$IP_CONFIG_NAME" \
  --subnet "$SUBNET_ID" \
  --private-ip-address-version IPv6 \
  --public-ip-address "$PIP6_NAME"

$ kubectl get nodes -o wide
NAME                             STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP             OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-default-bsqxr                Ready    <none>   172m    v1.33.7   172.16.1.10   2603:1020:1001:4::144   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-hvvft                Ready    <none>   173m    v1.33.7   172.16.1.6    <none>                  Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-40454228-vmss000000   Ready    <none>   5h19m   v1.33.7   172.16.1.4    <none>                  Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2