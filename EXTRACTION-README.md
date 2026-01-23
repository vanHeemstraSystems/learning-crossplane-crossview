# Learning Crossplane Crossview - Repository Archive

This archive contains all the files for the learning-crossplane-crossview repository, updated for **Crossplane 2.x**.

## Extraction Instructions

### Quick Start

```bash
# Navigate to your GitHub repositories directory
cd ~/github  # or wherever you keep your repos

# Extract the archive
tar -xzf learning-crossplane-crossview.tar.gz

# Navigate into the repository
cd learning-crossplane-crossview

# Verify the contents
ls -la
```

### Expected Directory Structure

After extraction, you should have:

```
learning-crossplane-crossview/
├── README.md                           # Main repository README
├── CROSSPLANE-2X-MIGRATION.md         # Migration guide for v2.x
├── docs/
│   ├── 01-prerequisites.md            # Prerequisites and setup
│   ├── 02-aks-setup.md                # AKS cluster setup
│   └── 03-crossplane-installation.md  # Crossplane installation guide
├── examples/
│   ├── sample-xrd/
│   │   ├── definition.yaml            # XRD using v2 API
│   │   ├── composition.yaml           # Composition using v1 API
│   │   └── xr.yaml                    # XR instance (replaces claim)
│   ├── azure-resources/
│   │   ├── resource-group.yaml
│   │   ├── storage-account.yaml
│   │   └── sql-database.yaml
│   └── multi-resource/
│       ├── app-platform-xrd.yaml
│       └── app-platform-xr.yaml
└── manifests/
    ├── namespace.yaml
    └── postgresql/
        └── deployment.yaml
```

## Initialize Git Repository (Optional)

If you want to track this as a Git repository:

```bash
cd learning-crossplane-crossview

# Initialize git
git init

# Create .gitignore
cat > .gitignore <<EOF
# Secrets
**/secrets.yaml
*.secret
*.key

# Kubeconfig
kubeconfig*
*.kubeconfig

# Environment files
.env
*.env

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db
EOF

# Add all files
git add .

# Initial commit
git commit -m "Initial commit: Crossplane 2.x learning repository"

# Add remote (replace with your GitHub repo URL)
git remote add origin https://github.com/vanHeemstraSystems/learning-crossplane-crossview.git

# Push to GitHub
git push -u origin main
```

## Quick Start Guide

1. **Read the migration guide first**: 
   ```bash
   cat CROSSPLANE-2X-MIGRATION.md
   ```

2. **Follow the documentation in order**:
   - `docs/01-prerequisites.md` - Set up your environment
   - `docs/02-aks-setup.md` - Create your AKS cluster
   - `docs/03-crossplane-installation.md` - Install Crossplane 2.x

3. **Try the examples**:
   ```bash
   # Apply the sample XRD
   kubectl apply -f examples/sample-xrd/definition.yaml
   
   # Apply the composition
   kubectl apply -f examples/sample-xrd/composition.yaml
   
   # Create an XR instance
   kubectl apply -f examples/sample-xrd/xr.yaml
   ```

## Key Changes in This Version

✅ **Crossplane 2.x Compatible**
- XRDs use `apiVersion: apiextensions.crossplane.io/v2`
- Compositions use `apiVersion: apiextensions.crossplane.io/v1`
- Claims removed - XRs can be namespace-scoped directly
- All examples updated for v2.x

✅ **Azure-Focused**
- All examples use Azure resources
- Azure provider configuration included
- AKS-specific guidance

✅ **Complete Documentation**
- Prerequisites guide
- AKS setup instructions  
- Crossplane installation guide
- Migration guide for v2.x changes

## Getting Help

- Read `CROSSPLANE-2X-MIGRATION.md` for understanding v2.x changes
- Check `docs/` directory for detailed guides
- Review examples in `examples/` directory
- Visit [Crossplane Documentation](https://docs.crossplane.io/)
- Join [Crossplane Slack](https://slack.crossplane.io/)

## Archive Information

- **Created**: January 23, 2026
- **Crossplane Version**: 2.1+
- **Format**: tar.gz (gzipped tarball)
- **Size**: ~18KB

---

Happy learning! 🚀
