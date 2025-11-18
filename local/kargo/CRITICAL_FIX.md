# CRITICAL FIX: Dual Kargo Installation

## Issue Found

During SRE review, discovered that Kargo was being installed **TWICE**:

1. **Fleet-based** (NEW): Via fleet-manifests/local/kargo/
2. **Script-based** (OLD): Via scripts/kargo_init.sh called from rancher-server.sh

## Root Cause

In `scripts/rancher-server.sh` line 314:
```bash
if [[ "${KARGO_ENABLED}" == "true" ]]; then
  bash /vagrant/scripts/kargo_init.sh  # This was still running!
fi
```

When `kargo.enabled: true` in vagrant-config.yml, it triggered BOTH installation paths.

## Impact

- Conflicting Helm releases
- Duplicate Project/Warehouse/Stages resources
- Two different secrets: git-repo-creds (script) vs git-credentials (Fleet)
- Unpredictable reconciliation behavior
- Fleet trying to manage resources created by script
- Source of truth unclear (Git vs script)

## Fix Applied

### 1. Modified `scripts/rancher-server.sh`

Changed condition to require explicit opt-in:
```bash
if [[ "${KARGO_ENABLED}" == "true" && "${KARGO_USE_SCRIPT}" == "true" ]]; then
  log "WARNING: Using deprecated script-based Kargo installation"
  bash /vagrant/scripts/kargo_init.sh
else
  log "Kargo managed by Fleet (see fleet-manifests/local/kargo/)"
fi
```

**Result:** Script path is now **disabled by default**

### 2. Updated `vagrant-config.yml`

Added documentation:
```yaml
kargo:
  # Kargo is now deployed via Fleet (see fleet-manifests/local/kargo/)
  # Setting enabled: true only controls secret provisioning
  enabled: true
  # use_script: true  # Uncomment to use deprecated script installation
```

**Result:** Clear separation of concerns - Fleet for infrastructure, enabled for secrets

### 3. Updated Documentation

- README.md: Clarified deployment flow
- SRE_REVIEW.md: Added P0 issue and fix
- Comments in scripts explaining the change

## Verification

After fix, provisioning should show:

```bash
# Should see Fleet deployment
kubectl get helmchart kargo -n kube-system

# Should see Fleet bundle
kubectl get bundles -n fleet-local | grep kargo

# Should NOT see script output
vagrant provision | grep "KARGO INSTALLED"  # Empty

# Only ONE secret
kubectl get secret -n hello-kargo  # Should see git-credentials only
```

## Migration Path

For users currently using script-based installation:

1. **One-time manual cleanup:**
   ```bash
   # Remove script-installed resources
   kubectl delete helmrelease kargo -n kube-system
   kubectl delete secret git-repo-creds -n hello-kargo
   ```

2. **Deploy via Fleet:**
   ```bash
   # Copy manifests to Fleet repo
   cp -r fleet-manifests/local/kargo /path/to/rancher-fleet-test/local/
   git add local/kargo/
   git commit -m "Migrate to Fleet-managed Kargo"
   git push
   ```

3. **Re-provision:**
   ```bash
   vagrant provision local-ctrl
   ```

4. **Verify:**
   ```bash
   # Check Fleet ownership
   kubectl get project hello-kargo -n hello-kargo -o yaml | grep -A5 metadata
   # Should see: fleet.cattle.io annotations
   ```

## Status

- **Priority:** P0 (Critical)
- **Fixed:** Yes
- **Tested:** Configuration verified
- **Impact:** Prevents dual installation on fresh deployments
- **Backward Compat:** Old script available via use_script: true flag

## Lessons Learned

1. **Always check existing provisioning paths** when adding new ones
2. **Feature flags should be explicit** (use_script vs just enabled)
3. **Document migration paths** when deprecating functionality
4. **Test both fresh and upgrade scenarios**
5. **SRE reviews catch production-breaking issues**

---

**Fixed by:** Senior SRE Review
**Date:** 2025-11-18
**Severity:** CRITICAL (would break production deployments)
**Resolution:** Script path disabled, Fleet is sole source of truth
