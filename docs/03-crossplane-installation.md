# Crossplane Installation on AKS

This guide walks you through installing Crossplane 2.x on your AKS cluster with the Azure provider.

## Prerequisites

Before installing Crossplane, ensure you have:
- An AKS cluster running (see [02-aks-setup.md](02-aks-setup.md))
- kubectl configured to access your cluster
- Helm 3.x installed
- Azure credentials for the provider

## Understanding Crossplane 2.x

### Key Changes in Crossplane 2.x

**Important**: Crossplane 2.x introduces significant changes:

1. **Claims Removed**: The Claim (XRC) concept has been removed. XRs can now be namespace-scoped directly.
2. **XRD API Version**: XRDs now use `apiVersion: apiextensions.crossplane.io/v2`
3. **Compositions Still v1**: Compositions continue to use `apiVersion: apiextensions.crossplane.io/v1`
4. **Namespace-scoped XRs**: XRs can be created in namespaces without needing Claims

### Resource Hierarchy in Crossplane 2.x

```
XRD (v2) - Defines the schema
    ↓
Composition (v1) - Defines the template
    ↓
XR - Instance (can be namespace or cluster scoped)
    ↓
Managed Resources - Actual cloud resources
```

## Step 1: Install Crossplane

### Add Crossplane Helm Repository

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

### Install Crossplane 2.x

```bash
# Install Crossplane
helm install crossplane \
  crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 1.15.0  # Latest stable version supporting v2 XRDs

# Wait for Crossplane to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/crossplane -n crossplane-system

kubectl wait --for=condition=available --timeout=300s \
  deployment/crossplane-rbac-manager -n crossplane-system
```

### Verify Installation

```bash
# Check Crossplane pods
kubectl get pods -n crossplane-system

# Expected output:
# NAME                                      READY   STATUS    RESTARTS   AGE
# crossplane-xxx                            1/1     Running   0          2m
# crossplane-rbac-manager-xxx               1/1     Running   0          2m

# Check Crossplane version
kubectl get deployment crossplane -n crossplane-system -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Step 2: Install Azure Provider

### Install the Provider Package

```bash
# Create provider configuration
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-upbound
spec:
  package: xpkg.upbound.io/upbound/provider-azure-upbound:v0.42.0
  packagePullPolicy: Always
EOF

# Wait for provider to be installed
kubectl wait --for=condition=healthy --timeout=300s \
  provider/provider-azure-upbound

# Check provider status
kubectl get providers
```

### Configure Provider Authentication

#### Option A: Service Principal (Development)

```bash
# Get your Azure credentials
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export TENANT_ID=$(az account show --query tenantId -o tsv)

# Create service principal
AZURE_CREDS=$(az ad sp create-for-rbac \
  --name "crossplane-provider" \
  --role Contributor \
  --scopes "/subscriptions/${SUBSCRIPTION_ID}" \
  --query '{clientId:appId, clientSecret:password}' \
  --output json)

export CLIENT_ID=$(echo $AZURE_CREDS | jq -r '.clientId')
export CLIENT_SECRET=$(echo $AZURE_CREDS | jq -r '.clientSecret')

# Create Kubernetes secret
kubectl create secret generic azure-creds \
  -n crossplane-system \
  --from-literal=credentials="$(cat <<EOF
{
  "clientId": "${CLIENT_ID}",
  "clientSecret": "${CLIENT_SECRET}",
  "subscriptionId": "${SUBSCRIPTION_ID}",
  "tenantId": "${TENANT_ID}"
}
EOF
)"

# Create ProviderConfig
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-creds
      key: credentials
EOF
```

#### Option B: Workload Identity (Production)

```bash
# Enable OIDC issuer (if not already done)
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get OIDC issuer URL
export OIDC_ISSUER=$(az aks show \
  --resource-group ${RESOURCE_GROUP} \
  --name ${CLUSTER_NAME} \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv)

# Create managed identity
az identity create \
  --resource-group ${RESOURCE_GROUP} \
  --name crossplane-identity

export IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group ${RESOURCE_GROUP} \
  --name crossplane-identity \
  --query clientId -o tsv)

export IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group ${RESOURCE_GROUP} \
  --name crossplane-identity \
  --query principalId -o tsv)

# Assign Contributor role
az role assignment create \
  --assignee ${IDENTITY_PRINCIPAL_ID} \
  --role Contributor \
  --scope /subscriptions/${SUBSCRIPTION_ID}

# Create federated credential
az identity federated-credential create \
  --name crossplane-federated-credential \
  --identity-name crossplane-identity \
  --resource-group ${RESOURCE_GROUP} \
  --issuer ${OIDC_ISSUER} \
  --subject system:serviceaccount:crossplane-system:provider-azure-upbound

# Create ProviderConfig for Workload Identity
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
  clientID: ${IDENTITY_CLIENT_ID}
  subscriptionID: ${SUBSCRIPTION_ID}
  tenantID: ${TENANT_ID}
EOF
```

## Step 3: Test Crossplane Installation

### Create a Test XRD (v2)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: xresourcegroups.azure.example.com
spec:
  group: azure.example.com
  names:
    kind: XResourceGroup
    plural: xresourcegroups
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              location:
                type: string
                default: eastus
          status:
            type: object
            properties:
              ready:
                type: boolean
EOF
```

### Create a Test Composition

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xresourcegroups.azure
spec:
  compositeTypeRef:
    apiVersion: azure.example.com/v1alpha1
    kind: XResourceGroup
  mode: Pipeline
  pipeline:
  - step: create-resource-group
    functionRef:
      name: function-patch-and-transform
    input:
      apiVersion: pt.fn.crossplane.io/v1beta1
      kind: Resources
      resources:
      - name: resource-group
        base:
          apiVersion: azure.upbound.io/v1beta1
          kind: ResourceGroup
          spec:
            forProvider:
              location: eastus
        patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.location
          toFieldPath: spec.forProvider.location
EOF
```

### Create a Test XR

```bash
cat <<EOF | kubectl apply -f -
apiVersion: azure.example.com/v1alpha1
kind: XResourceGroup
metadata:
  name: test-rg
  namespace: default
spec:
  location: eastus
  compositionRef:
    name: xresourcegroups.azure
EOF
```

### Verify Resources

```bash
# Check XR status
kubectl get xresourcegroups

# Check managed resource
kubectl get resourcegroups

# Describe the XR for details
kubectl describe xresourcegroup test-rg

# Check events
kubectl get events --sort-by='.lastTimestamp' | grep test-rg
```

### Cleanup Test Resources

```bash
kubectl delete xresourcegroup test-rg
kubectl delete composition xresourcegroups.azure
kubectl delete xrd xresourcegroups.azure.example.com
```

## Step 4: Install Composition Functions

Crossplane 2.x uses Functions for transformations. Install the patch-and-transform function:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.2.1
EOF

# Wait for function to be ready
kubectl wait --for=condition=healthy --timeout=300s \
  function/function-patch-and-transform
```

## Common Issues

### Provider Not Healthy

```bash
# Check provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-azure-upbound

# Check provider events
kubectl describe provider provider-azure-upbound
```

### Authentication Errors

```bash
# Verify secret exists
kubectl get secret azure-creds -n crossplane-system

# Verify ProviderConfig
kubectl get providerconfig default -o yaml

# Test Azure credentials manually
az login --service-principal \
  --username ${CLIENT_ID} \
  --password ${CLIENT_SECRET} \
  --tenant ${TENANT_ID}
```

### XRD Version Errors

If you see errors about XRD versions:

```bash
# Ensure you're using v2 for XRDs
apiVersion: apiextensions.crossplane.io/v2  # Correct for Crossplane 2.x
kind: CompositeResourceDefinition

# NOT v1 (old version)
apiVersion: apiextensions.crossplane.io/v1  # Only for Compositions
```

## Next Steps

Now that Crossplane is installed, proceed to:
- [04 - Crossview Deployment](04-crossview-deployment.md)

## Additional Resources

- [Crossplane Documentation](https://docs.crossplane.io/)
- [Azure Provider Docs](https://marketplace.upbound.io/providers/upbound/provider-azure-upbound)
- [Composition Functions](https://docs.crossplane.io/latest/concepts/composition-functions/)
