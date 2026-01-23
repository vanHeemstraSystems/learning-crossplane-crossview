# Crossplane 2.x Migration Guide

This guide explains the key changes in Crossplane 2.x and how they affect this learning repository.

## Key Changes in Crossplane 2.x

### 1. Claims Removed ❌

**What Changed:**
- The Claim (XRC) concept has been completely removed
- You now create XRs directly, which can be namespace-scoped or cluster-scoped

**Before (Crossplane 1.x):**
```yaml
# Claim (namespace-scoped)
apiVersion: database.example.com/v1alpha1
kind: PostgreSQLInstance  # Claim kind
metadata:
  namespace: production
  name: my-db
spec:
  size: large

# This would create a cluster-scoped XR automatically
```

**After (Crossplane 2.x):**
```yaml
# XR can be namespace-scoped directly
apiVersion: database.example.com/v1alpha1
kind: XPostgreSQLInstance  # XR kind
metadata:
  namespace: production  # XRs can now have namespaces!
  name: my-db
spec:
  size: large
```

### 2. XRD API Version Changed to v2 📝

**What Changed:**
- XRDs now use `apiVersion: apiextensions.crossplane.io/v2`
- The `claimNames` field has been removed from XRD spec

**Before (Crossplane 1.x):**
```yaml
apiVersion: apiextensions.crossplane.io/v1  # Old version
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.example.com
spec:
  group: example.com
  names:
    kind: XDatabase
    plural: xdatabases
  claimNames:  # This field is gone in v2
    kind: Database
    plural: databases
```

**After (Crossplane 2.x):**
```yaml
apiVersion: apiextensions.crossplane.io/v2  # New version
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.example.com
spec:
  group: example.com
  names:
    kind: XDatabase
    plural: xdatabases
  # No claimNames field!
```

### 3. Compositions Remain at v1 ✅

**Important:** Compositions still use `apiVersion: apiextensions.crossplane.io/v1`

```yaml
apiVersion: apiextensions.crossplane.io/v1  # Still v1!
kind: Composition
metadata:
  name: database-composition
spec:
  compositeTypeRef:
    apiVersion: example.com/v1alpha1
    kind: XDatabase
```

### 4. Namespace-Scoped XRs 🎯

**What's New:**
- XRs can now be created in namespaces without needing Claims
- Perfect for multi-tenant scenarios
- Each namespace can have its own XRs

**Example:**
```yaml
apiVersion: database.example.com/v1alpha1
kind: XPostgreSQLDatabase
metadata:
  name: app-db
  namespace: team-a  # Team A's namespace
spec:
  parameters:
    size: small

---
apiVersion: database.example.com/v1alpha1
kind: XPostgreSQLDatabase
metadata:
  name: app-db
  namespace: team-b  # Team B's namespace
spec:
  parameters:
    size: large
```

## Repository Updates

### Files Updated

All files in this repository have been updated for Crossplane 2.x:

✅ **README.md**
- Removed Claims references
- Updated resource hierarchy
- Added v2 XRD notes

✅ **docs/01-prerequisites.md**
- Updated Crossplane basics section
- Removed Claims from prerequisites

✅ **docs/03-crossplane-installation.md**
- Complete guide for Crossplane 2.x installation
- Explains XRD v2 vs Composition v1
- Updated examples

✅ **examples/sample-xrd/**
- `definition.yaml` - Uses v2 XRD API
- `composition.yaml` - Uses v1 Composition API
- `xr.yaml` - Replaces old `claim.yaml`

✅ **examples/azure-resources/**
- All examples use namespace-scoped XRs
- No more claim files

✅ **examples/multi-resource/**
- `app-platform-xrd.yaml` - v2 XRD
- `app-platform-xr.yaml` - Replaces claim

## Crossview Compatibility

Crossview is fully compatible with Crossplane 2.x:

- ✅ Displays XRDs (v2)
- ✅ Shows Compositions (v1)
- ✅ Visualizes namespace-scoped XRs
- ✅ Tracks managed resources
- ❌ No more "Claims" section (it's gone!)

### What You'll See in Crossview

**Resource Hierarchy:**
```
XRD (Definition)
  ↓
Composition (Template)
  ↓
XR (Instance) - can be in any namespace
  ↓
Managed Resources (Azure resources)
```

## Migration Checklist

If you're migrating from Crossplane 1.x to 2.x:

- [ ] Update all XRDs to use `apiVersion: apiextensions.crossplane.io/v2`
- [ ] Remove `claimNames` from all XRDs
- [ ] Replace Claim resources with namespace-scoped XRs
- [ ] Keep Compositions at `apiVersion: apiextensions.crossplane.io/v1`
- [ ] Update kubectl commands (no more `kubectl get claims`)
- [ ] Update documentation and examples
- [ ] Test all XRs create successfully
- [ ] Verify Crossview displays resources correctly

## Common Errors and Solutions

### Error: "claimNames not supported"

**Problem:**
```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
spec:
  claimNames:  # ERROR: Not valid in v2
    kind: Database
```

**Solution:** Remove the `claimNames` field entirely.

### Error: "Unknown API version"

**Problem:**
```yaml
apiVersion: apiextensions.crossplane.io/v1  # Wrong for XRDs in 2.x
kind: CompositeResourceDefinition
```

**Solution:** Use v2 for XRDs:
```yaml
apiVersion: apiextensions.crossplane.io/v2  # Correct
kind: CompositeResourceDefinition
```

### Error: "Resource not found" for Claims

**Problem:** Trying to use old claim commands:
```bash
kubectl get claims  # These don't exist anymore!
```

**Solution:** Use XR commands instead:
```bash
kubectl get xpostgresqldatabases -n production
kubectl describe xpostgresqldatabase my-db -n production
```

## Benefits of Crossplane 2.x

1. **Simpler Model**: No need to understand Claims vs XRs
2. **Direct Creation**: Create XRs directly in namespaces
3. **Better Multi-tenancy**: Each namespace can have its own XRs
4. **Clearer Hierarchy**: Fewer resource types to manage
5. **Less Confusion**: One way to create resources, not two

## Resources

- [Crossplane 2.x Release Notes](https://github.com/crossplane/crossplane/releases)
- [XRD v2 Documentation](https://docs.crossplane.io/)
- [Migration Guide](https://docs.crossplane.io/latest/concepts/composite-resources/)

## Questions?

If you encounter issues with Crossplane 2.x:

1. Check the [Troubleshooting Guide](docs/08-troubleshooting.md)
2. Review [Crossplane Documentation](https://docs.crossplane.io/)
3. Ask in [Crossplane Slack](https://slack.crossplane.io/)
4. Open an issue in this repository

---

**Last Updated:** January 2026 for Crossplane 2.1+
