# Learning Crossplane Crossview

A comprehensive guide to deploying and using Crossview - a modern dashboard for visualizing and managing Crossplane resources - on Azure Kubernetes Service (AKS).

> **📢 Important:** This repository uses **Crossplane 2.x** (2.1+). If you're familiar with older versions, please read [CROSSPLANE-2X-MIGRATION.md](CROSSPLANE-2X-MIGRATION.md) for key changes, especially the removal of Claims and the new XRD v2 API.

- [References](./REFERENCES.md)

## Overview

**Crossview** is a React-based web dashboard that provides visual management and monitoring of Crossplane resources in Kubernetes. This repository serves as a learning resource for understanding, deploying, and using Crossview specifically on AKS clusters.

### What is Crossview?

Crossview is an open-source tool that helps you:
- **Visualize** Crossplane Composite Resources (XRs), XRDs, and Compositions
- **Monitor** resource health and status in real-time
- **Search** across all Crossplane resources efficiently
- **Manage** multi-cluster Crossplane deployments
- **Understand** resource relationships and dependencies

### Why Use Crossview?

- **Better Understanding**: Visual interface makes Crossplane concepts easier to grasp
- **Faster Troubleshooting**: Quickly identify issues with resource health and relationships
- **Team Collaboration**: Easier onboarding for team members new to Crossplane
- **Specialized for Crossplane**: Unlike generic Kubernetes dashboards, Crossview understands Crossplane's unique resource model

## Repository Structure

```
learning-crossplane-crossview/
├── README.md                           # This file
├── docs/
│   ├── 01-prerequisites.md            # Prerequisites and setup requirements
│   ├── 02-aks-setup.md               # Setting up AKS cluster
│   ├── 03-crossplane-installation.md # Installing Crossplane on AKS
│   ├── 04-crossview-deployment.md    # Deploying Crossview
│   ├── 05-configuration.md           # Configuring Crossview
│   ├── 06-using-crossview.md         # Using Crossview UI
│   ├── 07-advanced-features.md       # Advanced features and tips
│   ├── 08-troubleshooting.md         # Common issues and solutions
│   └── 09-best-practices.md          # Best practices and recommendations
├── manifests/
│   ├── namespace.yaml                # Crossview namespace
│   ├── postgresql/
│   │   ├── pvc.yaml                  # Persistent volume claim for PostgreSQL
│   │   ├── deployment.yaml           # PostgreSQL deployment
│   │   └── service.yaml              # PostgreSQL service
│   ├── crossview/
│   │   ├── configmap.yaml           # Crossview configuration
│   │   ├── secrets.yaml.example     # Example secrets (DO NOT commit real secrets)
│   │   ├── deployment.yaml          # Crossview deployment
│   │   ├── service.yaml             # Crossview service
│   │   └── ingress.yaml             # Ingress configuration
│   └── rbac/
│       ├── serviceaccount.yaml      # Service account for Crossview
│       ├── clusterrole.yaml         # Cluster role with permissions
│       └── clusterrolebinding.yaml  # Cluster role binding
├── helm/
│   ├── values-aks.yaml              # Helm values customized for AKS
│   ├── values-dev.yaml              # Development environment values
│   └── values-prod.yaml             # Production environment values
├── examples/
│   ├── sample-xrd/
│   │   ├── definition.yaml          # Sample XRD
│   │   ├── composition.yaml         # Sample Composition
│   │   └── xr.yaml                  # Sample XR instance
│   ├── azure-resources/
│   │   ├── resource-group.yaml     # Azure Resource Group example
│   │   ├── storage-account.yaml    # Azure Storage Account example
│   │   └── sql-database.yaml       # Azure SQL Database example
│   └── multi-resource/
│       ├── app-platform-xrd.yaml   # Complex XRD example
│       └── app-platform-xr.yaml    # Complex XR instance
├── scripts/
│   ├── setup-aks.sh                # Automated AKS setup script
│   ├── install-crossplane.sh       # Crossplane installation script
│   ├── deploy-crossview.sh         # Crossview deployment script
│   ├── cleanup.sh                  # Cleanup resources script
│   └── port-forward.sh             # Port forwarding helper
└── tests/
    ├── verify-crossplane.sh        # Verify Crossplane installation
    ├── verify-crossview.sh         # Verify Crossview deployment
    └── smoke-tests.sh              # Basic smoke tests
```

## Quick Start

### Prerequisites

- Azure subscription
- Azure CLI installed and configured
- kubectl installed
- Helm 3.x installed
- Basic understanding of Kubernetes and Crossplane

### 1. Create AKS Cluster

```bash
# Set variables
export RESOURCE_GROUP="rg-crossplane-learning"
export CLUSTER_NAME="aks-crossplane-dev"
export LOCATION="westeurope"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create AKS cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

### 2. Install Crossplane

```bash
# Add Crossplane Helm repository
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Install Crossplane
helm install crossplane \
  crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace

# Wait for Crossplane to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/crossplane -n crossplane-system
```

### 3. Install Azure Provider

```bash
# Install Azure provider
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-upbound
spec:
  package: xpkg.upbound.io/upbound/provider-azure:v0.42.0
EOF

# Wait for provider to be healthy
kubectl wait --for=condition=healthy --timeout=300s \
  provider/provider-azure-upbound
```

### 4. Deploy Crossview

```bash
# Add Crossview Helm repository
helm repo add crossview https://corpobit.github.io/crossview
helm repo update

# Create namespace
kubectl create namespace crossview

# Install Crossview with Helm
helm install crossview crossview/crossview \
  --namespace crossview \
  --set secrets.dbPassword=$(openssl rand -base64 32) \
  --set secrets.sessionSecret=$(openssl rand -base64 32) \
  --set service.type=LoadBalancer

# Wait for Crossview to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/crossview -n crossview

# Get the external IP
kubectl get service crossview -n crossview
```

### 5. Access Crossview

```bash
# Option 1: Port forwarding (for development)
kubectl port-forward -n crossview svc/crossview 3001:3001

# Then access at http://localhost:3001

# Option 2: LoadBalancer (if configured)
# Get external IP from the service
EXTERNAL_IP=$(kubectl get svc crossview -n crossview -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Access Crossview at http://$EXTERNAL_IP:3001"
```

## Learning Path

Follow this recommended learning path to master Crossview on AKS:

### Week 1: Foundations
1. **Day 1-2**: Read [Prerequisites](docs/01-prerequisites.md) and [AKS Setup](docs/02-aks-setup.md)
2. **Day 3-4**: Complete [Crossplane Installation](docs/03-crossplane-installation.md)
3. **Day 5-7**: Deploy Crossview following [Crossview Deployment](docs/04-crossview-deployment.md)

### Week 2: Configuration and Usage
1. **Day 1-3**: Configure Crossview using [Configuration Guide](docs/05-configuration.md)
2. **Day 4-5**: Learn the UI with [Using Crossview](docs/06-using-crossview.md)
3. **Day 6-7**: Explore [Advanced Features](docs/07-advanced-features.md)

### Week 3: Practice and Mastery
1. **Day 1-3**: Work through examples in `examples/` directory
2. **Day 4-5**: Practice troubleshooting using [Troubleshooting Guide](docs/08-troubleshooting.md)
3. **Day 6-7**: Implement [Best Practices](docs/09-best-practices.md)

## Key Concepts

### Crossplane Resources Visualized in Crossview

1. **Composite Resource Definitions (XRDs)**
   - Define the schema for your custom APIs
   - Visible in Crossview's "Definitions" section
   - Uses `apiVersion: apiextensions.crossplane.io/v2` in Crossplane 2.x

2. **Compositions**
   - Templates that define how to compose resources
   - View relationships between XRDs and Compositions
   - Uses `apiVersion: apiextensions.crossplane.io/v1`

3. **Composite Resources (XRs)**
   - Custom resources created from XRDs
   - Can be namespace-scoped or cluster-scoped in Crossplane 2.x
   - See real-time status and health

4. **Managed Resources**
   - Actual cloud resources (Azure VMs, Storage Accounts, etc.)
   - Monitor provisioning status

### Crossview Dashboard Features

- **Resource Browser**: Navigate all Crossplane resources by type
- **Search**: Find resources by name, namespace, or labels
- **Health Dashboard**: Real-time status of all resources
- **Resource Details**: Deep dive into individual resources
- **Relationship View**: Visualize dependencies between resources
- **Multi-Cluster**: Switch between multiple Kubernetes contexts

## Azure-Specific Considerations

### AKS Integration

- **Managed Identity**: Use Azure AD Workload Identity for secure provider authentication
- **Azure Key Vault**: Store sensitive credentials securely
- **Azure Monitor**: Integrate Crossview logs with Azure Monitor
- **Network Policies**: Secure Crossview with Azure Network Policies

### Cost Optimization

- Use appropriate node sizes (Standard_D4s_v3 recommended for development)
- Enable cluster autoscaling
- Use Azure Disk for PostgreSQL persistent storage
- Consider Azure Database for PostgreSQL for production deployments

## Common Use Cases

### 1. Platform Engineering
```bash
# Create a platform API for databases
kubectl apply -f examples/azure-resources/sql-database.yaml

# View in Crossview to understand the resource composition
```

### 2. Multi-Environment Management
```bash
# Deploy to development
helm install crossview crossview/crossview -f helm/values-dev.yaml

# Deploy to production
helm install crossview crossview/crossview -f helm/values-prod.yaml
```

### 3. Troubleshooting Failed Resources
- Use Crossview to identify which managed resources are failing
- View events and conditions in the resource detail view
- Check composition logs for debugging

## Contributing to Learning

This is a learning repository. Contributions are welcome!

### How to Contribute

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/new-tutorial`)
3. Add your content (documentation, examples, or scripts)
4. Commit your changes (`git commit -m 'Add tutorial on X'`)
5. Push to the branch (`git push origin feature/new-tutorial`)
6. Open a Pull Request

### Contribution Ideas

- Additional Azure resource examples
- Advanced Crossview configurations
- Integration guides (Grafana, Prometheus, etc.)
- Video tutorials
- Real-world use cases

## Resources

### Official Documentation
- [Crossview GitHub](https://github.com/corpobit/crossview)
- [Crossplane Documentation](https://docs.crossplane.io/)
- [Azure Provider Documentation](https://marketplace.upbound.io/providers/upbound/provider-azure)
- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)

### Community
- [Crossplane Slack](https://slack.crossplane.io/) - #general, #help channels
- [Crossview Issues](https://github.com/corpobit/crossview/issues)
- [Crossplane Community](https://github.com/crossplane/crossplane#get-involved)

### Related Projects
- [Komoplane](https://github.com/komodorio/komoplane) - Alternative Crossplane troubleshooting tool
- [Upbound Cloud](https://www.upbound.io/) - Managed Crossplane control planes
- [Provider Azure](https://github.com/upbound/provider-azure) - Azure provider for Crossplane

## Troubleshooting

### Quick Diagnostics

```bash
# Check Crossview pods
kubectl get pods -n crossview

# Check Crossview logs
kubectl logs -n crossview deployment/crossview

# Check PostgreSQL connectivity
kubectl exec -it -n crossview deployment/crossview -- \
  pg_isready -h postgresql -p 5432

# Verify Crossplane installation
kubectl get providers
kubectl get compositeresourcedefinitions

# Test Crossview API
kubectl port-forward -n crossview svc/crossview 3001:3001
curl http://localhost:3001/api/health
```

### Common Issues

1. **Crossview pod won't start**: Check PostgreSQL is running and credentials are correct
2. **Can't see resources**: Verify RBAC permissions are configured correctly
3. **Performance issues**: Increase resource limits or use Azure Database for PostgreSQL
4. **Network connectivity**: Check network policies and firewall rules

See [Troubleshooting Guide](docs/08-troubleshooting.md) for detailed solutions.

## Cleanup

To remove all resources:

```bash
# Delete Crossview
helm uninstall crossview -n crossview
kubectl delete namespace crossview

# Delete Crossplane
helm uninstall crossplane -n crossplane-system
kubectl delete namespace crossplane-system

# Delete AKS cluster
az aks delete --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --yes --no-wait

# Delete resource group (includes all resources)
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

## License

This learning repository is provided as-is for educational purposes.

- Crossview is licensed under Apache License 2.0
- Crossplane is licensed under Apache License 2.0

## Acknowledgments

- [Corpobit](https://corpobit.com) for creating Crossview
- [Crossplane Community](https://www.crossplane.io/) for the amazing control plane framework
- [Upbound](https://www.upbound.io/) for Provider development and support

## Next Steps

1. Clone this repository
2. Start with [Prerequisites](docs/01-prerequisites.md)
3. Follow the Quick Start guide above
4. Work through the examples
5. Join the community and share your learnings!

---

**Questions or Issues?** Open an issue in this repository or reach out to the community channels listed above.

**Happy Learning! 🚀**
