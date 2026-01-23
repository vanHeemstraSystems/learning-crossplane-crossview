# AKS Setup

This guide walks you through creating and configuring an Azure Kubernetes Service (AKS) cluster optimized for running Crossplane and Crossview.

## Overview

We'll create an AKS cluster with the following specifications:
- **Purpose**: Crossplane control plane with Crossview dashboard
- **Nodes**: 3 worker nodes (autoscaling enabled)
- **Node Size**: Standard_D4s_v3 (4 vCPU, 16GB RAM)
- **Networking**: Azure CNI (recommended for production)
- **Identity**: Managed Identity enabled
- **Monitoring**: Azure Monitor for containers (optional)

## Step 1: Set Environment Variables

Define your cluster configuration:

```bash
# Project naming
export PROJECT_NAME="crossplane-learning"
export ENVIRONMENT="dev"  # dev, staging, prod

# Azure configuration
export LOCATION="westeurope"  # Choose closest region
export RESOURCE_GROUP="rg-${PROJECT_NAME}-${ENVIRONMENT}"
export CLUSTER_NAME="aks-${PROJECT_NAME}-${ENVIRONMENT}"

# AKS configuration
export NODE_COUNT=3
export NODE_SIZE="Standard_D4s_v3"
export K8S_VERSION="1.28"  # Or latest stable

# Network configuration (optional - for Azure CNI)
export VNET_NAME="vnet-${PROJECT_NAME}"
export SUBNET_NAME="subnet-aks-nodes"
export VNET_ADDRESS_PREFIX="10.0.0.0/16"
export SUBNET_ADDRESS_PREFIX="10.0.1.0/24"

# Save to file for future use
cat > cluster-config.env <<EOF
PROJECT_NAME=${PROJECT_NAME}
ENVIRONMENT=${ENVIRONMENT}
LOCATION=${LOCATION}
RESOURCE_GROUP=${RESOURCE_GROUP}
CLUSTER_NAME=${CLUSTER_NAME}
NODE_COUNT=${NODE_COUNT}
NODE_SIZE=${NODE_SIZE}
K8S_VERSION=${K8S_VERSION}
EOF
```

## Step 2: Create Resource Group

```bash
# Create resource group
az group create \
  --name ${RESOURCE_GROUP} \
  --location ${LOCATION} \
  --tags \
    environment=${ENVIRONMENT} \
    project=${PROJECT_NAME} \
    managed-by=terraform-or-manual

# Verify creation
az group show --name ${RESOURCE_GROUP} -o table
```

## Step 3: Create AKS Cluster

### Option A: Quick Start (Kubenet Networking)

Fastest way to get started, suitable for development:

```bash
az aks create \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --location ${LOCATION} \
  --kubernetes-version ${K8S_VERSION} \
  --node-count ${NODE_COUNT} \
  --node-vm-size ${NODE_SIZE} \
  --enable-managed-identity \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5 \
  --network-plugin kubenet \
  --generate-ssh-keys \
  --tags \
    environment=${ENVIRONMENT} \
    project=${PROJECT_NAME}

# This takes 5-10 minutes
```

### Option B: Production Setup (Azure CNI)

Recommended for production environments:

```bash
# Create virtual network
az network vnet create \
  --resource-group ${RESOURCE_GROUP} \
  --name ${VNET_NAME} \
  --address-prefix ${VNET_ADDRESS_PREFIX} \
  --subnet-name ${SUBNET_NAME} \
  --subnet-prefix ${SUBNET_ADDRESS_PREFIX}

# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group ${RESOURCE_GROUP} \
  --vnet-name ${VNET_NAME} \
  --name ${SUBNET_NAME} \
  --query id -o tsv)

# Create AKS cluster with Azure CNI
az aks create \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --location ${LOCATION} \
  --kubernetes-version ${K8S_VERSION} \
  --node-count ${NODE_COUNT} \
  --node-vm-size ${NODE_SIZE} \
  --enable-managed-identity \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5 \
  --network-plugin azure \
  --vnet-subnet-id ${SUBNET_ID} \
  --docker-bridge-address 172.17.0.1/16 \
  --service-cidr 10.2.0.0/16 \
  --dns-service-ip 10.2.0.10 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --tags \
    environment=${ENVIRONMENT} \
    project=${PROJECT_NAME}

# This takes 10-15 minutes
```

### Verify Cluster Creation

```bash
# Check cluster status
az aks show \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --query "provisioningState" -o tsv

# Should output: Succeeded
```

## Step 4: Get Cluster Credentials

```bash
# Download kubectl credentials
az aks get-credentials \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --overwrite-existing

# Verify connection
kubectl cluster-info

# Check nodes
kubectl get nodes

# Expected output:
# NAME                                STATUS   ROLES   AGE   VERSION
# aks-nodepool1-12345678-vmss000000   Ready    agent   5m    v1.28.x
# aks-nodepool1-12345678-vmss000001   Ready    agent   5m    v1.28.x
# aks-nodepool1-12345678-vmss000002   Ready    agent   5m    v1.28.x
```

## Step 5: Configure Cluster

### Enable Workload Identity (Recommended for Production)

Workload Identity provides secure authentication for Crossplane:

```bash
# Enable OIDC issuer
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get OIDC issuer URL (save for Crossplane provider config)
export OIDC_ISSUER=$(az aks show \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

echo "OIDC Issuer: ${OIDC_ISSUER}"
```

### Create Storage Class for PostgreSQL

```bash
# Create storage class for PostgreSQL persistent volumes
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF

# Verify
kubectl get storageclass
```

### Install Ingress Controller (Optional)

For production access to Crossview via Ingress:

```bash
# Add nginx-ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install nginx-ingress
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

# Wait for external IP
kubectl get service -n ingress-nginx nginx-ingress-ingress-nginx-controller --watch

# Get the external IP
export INGRESS_IP=$(kubectl get service -n ingress-nginx nginx-ingress-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ingress IP: ${INGRESS_IP}"
```

## Step 6: Configure Azure AD Integration for Crossplane

Create a service principal or managed identity for Crossplane to manage Azure resources.

### Option A: Service Principal (Quick Start)

```bash
# Get subscription ID
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Create service principal
az ad sp create-for-rbac \
  --name "sp-${CLUSTER_NAME}-crossplane" \
  --role Contributor \
  --scopes /subscriptions/${SUBSCRIPTION_ID}

# Save the output:
# {
#   "appId": "YOUR_APP_ID",
#   "displayName": "sp-aks-crossplane-learning-dev-crossplane",
#   "password": "YOUR_PASSWORD",
#   "tenant": "YOUR_TENANT_ID"
# }

# Create Kubernetes secret for Crossplane
kubectl create secret generic azure-provider-creds \
  --namespace crossplane-system \
  --from-literal=credentials="{
  \"clientId\": \"YOUR_APP_ID\",
  \"clientSecret\": \"YOUR_PASSWORD\",
  \"subscriptionId\": \"${SUBSCRIPTION_ID}\",
  \"tenantId\": \"YOUR_TENANT_ID\"
}" \
  --dry-run=client -o yaml > azure-creds-secret.yaml

# Note: We'll apply this secret after installing Crossplane
```

### Option B: Managed Identity with Workload Identity (Production)

```bash
# Get AKS managed identity
export AKS_IDENTITY_CLIENT_ID=$(az aks show \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --query identityProfile.kubeletidentity.clientId -o tsv)

# Create user-assigned managed identity for Crossplane
az identity create \
  --resource-group ${RESOURCE_GROUP} \
  --name "id-${CLUSTER_NAME}-crossplane"

export CROSSPLANE_IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group ${RESOURCE_GROUP} \
  --name "id-${CLUSTER_NAME}-crossplane" \
  --query clientId -o tsv)

export CROSSPLANE_IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group ${RESOURCE_GROUP} \
  --name "id-${CLUSTER_NAME}-crossplane" \
  --query principalId -o tsv)

# Grant Contributor role to managed identity
az role assignment create \
  --assignee ${CROSSPLANE_IDENTITY_PRINCIPAL_ID} \
  --role Contributor \
  --scope /subscriptions/${SUBSCRIPTION_ID}

# Save for later use
echo "CROSSPLANE_IDENTITY_CLIENT_ID=${CROSSPLANE_IDENTITY_CLIENT_ID}" >> cluster-config.env
```

## Step 7: Verify Cluster Setup

Run these verification commands:

```bash
# Check cluster health
kubectl get --raw='/readyz?verbose'

# Check node status
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check available storage classes
kubectl get storageclass

# Check cluster info
kubectl cluster-info dump > cluster-info.txt

# View cluster details
az aks show \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  -o json > cluster-details.json
```

## Step 8: Install Monitoring (Optional but Recommended)

### Enable Azure Monitor for Containers

```bash
# Enable monitoring addon (if not already enabled)
az aks enable-addons \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --addons monitoring

# Create Log Analytics workspace (if needed)
az monitor log-analytics workspace create \
  --resource-group ${RESOURCE_GROUP} \
  --workspace-name "law-${CLUSTER_NAME}"

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group ${RESOURCE_GROUP} \
  --workspace-name "law-${CLUSTER_NAME}" \
  --query id -o tsv)

# Link AKS to workspace
az aks enable-addons \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --addons monitoring \
  --workspace-resource-id ${WORKSPACE_ID}
```

## Cluster Configuration Summary

Save your configuration for reference:

```bash
cat > cluster-summary.txt <<EOF
Cluster Configuration Summary
============================

Cluster Name: ${CLUSTER_NAME}
Resource Group: ${RESOURCE_GROUP}
Location: ${LOCATION}
Kubernetes Version: ${K8S_VERSION}
Node Count: ${NODE_COUNT}
Node Size: ${NODE_SIZE}

Connection Command:
az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER_NAME}

Subscription ID: ${SUBSCRIPTION_ID}
OIDC Issuer: ${OIDC_ISSUER}

Ingress IP: ${INGRESS_IP}

Created: $(date)
EOF

cat cluster-summary.txt
```

## Common Issues and Solutions

### Issue: Cluster Creation Fails

**Error**: "Insufficient quota"

**Solution**:
```bash
# Check quota
az vm list-usage --location ${LOCATION} -o table | grep "Standard DSv3 Family"

# Request quota increase
# Go to Azure Portal > Subscriptions > Usage + quotas
```

### Issue: Cannot Connect to Cluster

**Error**: "Unable to connect to the server"

**Solution**:
```bash
# Re-download credentials
az aks get-credentials \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --overwrite-existing

# Check kubectl config
kubectl config view
kubectl config current-context
```

### Issue: Nodes Not Ready

**Solution**:
```bash
# Check node status
kubectl get nodes -o wide

# Describe problematic node
kubectl describe node NODE_NAME

# Check system pods
kubectl get pods -n kube-system
```

## Cost Optimization

### Auto-shutdown for Development

```bash
# Stop cluster (deallocate nodes)
az aks stop --resource-group ${RESOURCE_GROUP} --name ${CLUSTER_NAME}

# Start cluster
az aks start --resource-group ${RESOURCE_GROUP} --name ${CLUSTER_NAME}
```

### Use Spot Instances for Non-Production

```bash
# Add spot node pool
az aks nodepool add \
  --resource-group ${RESOURCE_GROUP} \
  --cluster-name ${CLUSTER_NAME} \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3 \
  --node-vm-size Standard_D4s_v3
```

## Cleanup Instructions

To delete the cluster and all resources:

```bash
# Delete AKS cluster
az aks delete \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --yes \
  --no-wait

# Delete resource group (includes all resources)
az group delete \
  --name ${RESOURCE_GROUP} \
  --yes \
  --no-wait

# Remove kubectl context
kubectl config delete-context ${CLUSTER_NAME}
```

## Next Steps

Now that your AKS cluster is ready, proceed to:
- [03 - Crossplane Installation](03-crossplane-installation.md)

## Additional Resources

- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- [AKS Networking Concepts](https://docs.microsoft.com/en-us/azure/aks/concepts-network)
- [AKS Security Best Practices](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security)
- [Azure CLI AKS Reference](https://docs.microsoft.com/en-us/cli/azure/aks)
