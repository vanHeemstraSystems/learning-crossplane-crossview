# Prerequisites

Before deploying Crossview on AKS, ensure you have the following tools and knowledge.

## Required Tools

### 1. Azure CLI

**Version Required**: 2.50.0 or later

**Installation**:

**macOS**:
```bash
brew update && brew install azure-cli
```

**Linux**:
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Windows**:
Download and install from [Azure CLI Installer](https://aka.ms/installazurecliwindows)

**Verify Installation**:
```bash
az --version
az login
```

### 2. kubectl

**Version Required**: 1.25.0 or later (should match your AKS version ±1)

**Installation**:

**macOS**:
```bash
brew install kubectl
```

**Linux**:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Windows**:
```powershell
choco install kubernetes-cli
```

**Verify Installation**:
```bash
kubectl version --client
```

### 3. Helm

**Version Required**: 3.10.0 or later

**Installation**:

**macOS**:
```bash
brew install helm
```

**Linux**:
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Windows**:
```powershell
choco install kubernetes-helm
```

**Verify Installation**:
```bash
helm version
```

### 4. Git

**Version Required**: 2.30.0 or later

**Installation**:

**macOS**:
```bash
brew install git
```

**Linux**:
```bash
sudo apt-get install git  # Debian/Ubuntu
sudo yum install git       # RHEL/CentOS
```

**Windows**:
Download from [git-scm.com](https://git-scm.com/downloads)

**Verify Installation**:
```bash
git --version
```

### 5. Optional but Recommended

#### jq (JSON processor)
```bash
# macOS
brew install jq

# Linux
sudo apt-get install jq

# Windows
choco install jq
```

#### yq (YAML processor)
```bash
# macOS
brew install yq

# Linux
sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
sudo chmod +x /usr/local/bin/yq

# Windows
choco install yq
```

#### k9s (Kubernetes CLI UI)
```bash
# macOS
brew install k9s

# Linux
curl -sS https://webinstall.dev/k9s | bash

# Windows
choco install k9s
```

## Azure Requirements

### 1. Azure Subscription

You need an active Azure subscription with:
- Permissions to create resource groups
- Permissions to create AKS clusters
- Permissions to create Azure AD service principals (for Crossplane provider authentication)

**Verify Access**:
```bash
az account show
az account list-locations -o table
```

### 2. Resource Quotas

Ensure you have sufficient quotas in your subscription:

- **vCPUs**: At least 12 vCPUs for Standard_D4s_v3 nodes (3 nodes × 4 vCPUs)
- **Public IP Addresses**: At least 1 for LoadBalancer
- **Storage**: 100GB for persistent volumes

**Check Quotas**:
```bash
az vm list-usage --location westeurope -o table
```

### 3. Azure Service Principal (for Crossplane)

Create a service principal for Crossplane to manage Azure resources:

```bash
# Create service principal
az ad sp create-for-rbac \
  --name "crossplane-sp" \
  --role Contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID

# Save the output - you'll need:
# - appId (client ID)
# - password (client secret)
# - tenant
```

**Security Best Practice**: Use managed identities (Azure AD Workload Identity) in production instead of service principals.

## Knowledge Prerequisites

### 1. Kubernetes Fundamentals

You should understand:
- Pods, Deployments, Services
- ConfigMaps and Secrets
- Namespaces
- RBAC (Role-Based Access Control)
- Persistent Volumes and Claims
- Ingress controllers

**Resources to Learn**:
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [Learn Kubernetes Basics - Microsoft Learn](https://learn.microsoft.com/en-us/training/paths/intro-to-kubernetes-on-azure/)

### 2. Crossplane Basics

You should understand:
- What Crossplane is and why it's used
- Composite Resources (XR) - can be namespace-scoped or cluster-scoped in v2.x
- Composite Resource Definitions (XRD) - use v2 API
- Compositions - define resource templates
- Providers

**Resources to Learn**:
- [Crossplane Documentation](https://docs.crossplane.io/)
- [Crossplane Concepts](https://docs.crossplane.io/latest/concepts/)

### 3. Azure Fundamentals

You should understand:
- Azure Resource Groups
- Azure networking basics
- Azure Kubernetes Service (AKS)
- Azure identity and access management

**Resources to Learn**:
- [Azure Fundamentals](https://learn.microsoft.com/en-us/training/paths/azure-fundamentals/)
- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)

### 4. Helm Basics

You should understand:
- What Helm is
- Helm charts
- Helm values
- Helm repositories

**Resources to Learn**:
- [Helm Documentation](https://helm.sh/docs/)
- [Helm Quickstart](https://helm.sh/docs/intro/quickstart/)

## System Requirements

### Local Development Machine

**Minimum Requirements**:
- **CPU**: 2 cores
- **RAM**: 8 GB
- **Disk**: 20 GB free space
- **OS**: macOS 11+, Ubuntu 20.04+, Windows 10+

**Recommended Requirements**:
- **CPU**: 4 cores
- **RAM**: 16 GB
- **Disk**: 50 GB free space (SSD)
- **OS**: Latest stable version

### AKS Cluster

**Development Environment**:
- **Nodes**: 3
- **Node Size**: Standard_D4s_v3 (4 vCPU, 16 GB RAM)
- **Kubernetes Version**: 1.27 or later
- **Total vCPUs**: 12
- **Total RAM**: 48 GB

**Production Environment**:
- **Nodes**: 3-5 (with autoscaling)
- **Node Size**: Standard_D8s_v3 or larger
- **Kubernetes Version**: Latest stable
- **Availability Zones**: Enabled
- **Azure CNI**: Enabled (recommended)

## Network Requirements

### Required Outbound Connectivity

Your AKS cluster needs access to:

1. **Container Registries**:
   - `ghcr.io` (GitHub Container Registry)
   - `docker.io` (Docker Hub)
   - `mcr.microsoft.com` (Microsoft Container Registry)

2. **Helm Repositories**:
   - `https://charts.crossplane.io/stable`
   - `https://corpobit.github.io/crossview`

3. **Azure Services**:
   - Azure Resource Manager
   - Azure Active Directory
   - Azure Key Vault (if used)

4. **Package Registries**:
   - `https://xpkg.upbound.io` (Crossplane packages)

### Firewall Rules

If behind a corporate firewall, ensure these domains are allowed:

```
*.azure.com
*.azurecr.io
*.microsoft.com
*.githubusercontent.com
*.docker.io
*.crossplane.io
*.corpobit.com
```

## Cost Estimation

### AKS Cluster (Development)

**Monthly Costs** (approximate, West Europe):
- **3× Standard_D4s_v3 nodes**: ~€300-400/month
- **Load Balancer**: ~€20/month
- **Managed Disks (100GB)**: ~€10/month
- **Total**: ~€330-430/month

**Cost Optimization Tips**:
1. Use Azure Dev/Test pricing
2. Shut down cluster when not in use
3. Use spot instances for non-production
4. Enable cluster autoscaler

### Production Considerations

Additional costs for production:
- **Azure Database for PostgreSQL**: ~€100-300/month
- **Azure Key Vault**: ~€5/month
- **Azure Monitor**: ~€10-50/month (depending on usage)
- **Larger node pools**: Scale accordingly

**Cost Calculator**:
Use the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) for accurate estimates.

## Pre-flight Checklist

Before proceeding to AKS setup, verify:

- [ ] Azure CLI installed and logged in
- [ ] kubectl installed and working
- [ ] Helm 3.x installed
- [ ] Git installed
- [ ] Azure subscription access verified
- [ ] Sufficient Azure quotas available
- [ ] Service principal created (or plan to use managed identity)
- [ ] Basic Kubernetes knowledge confirmed
- [ ] Basic Crossplane concepts understood
- [ ] Network connectivity requirements met
- [ ] Cost estimates reviewed and approved

## Troubleshooting Prerequisites

### Azure CLI Login Issues

```bash
# Clear cached credentials
az account clear

# Login interactively
az login

# Login with service principal
az login --service-principal \
  --username APP_ID \
  --password PASSWORD \
  --tenant TENANT_ID
```

### kubectl Connection Issues

```bash
# Verify kubeconfig
kubectl config view

# Test cluster connectivity
kubectl cluster-info

# Check current context
kubectl config current-context

# Switch context if needed
kubectl config use-context CONTEXT_NAME
```

### Helm Issues

```bash
# Verify Helm installation
helm version

# Update Helm repositories
helm repo update

# List installed releases
helm list --all-namespaces
```

## Next Steps

Once all prerequisites are met, proceed to:
- [02 - AKS Setup](02-aks-setup.md)

## Additional Resources

- [Azure CLI Documentation](https://docs.microsoft.com/en-us/cli/azure/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Helm Documentation](https://helm.sh/docs/)
- [Azure Free Account](https://azure.microsoft.com/en-us/free/) (if you don't have a subscription)
