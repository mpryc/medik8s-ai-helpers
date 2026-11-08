---
name: medik8s-certsuite-validation
description: Validate Medik8s operator certsuite exceptions documentation against actual test results on OCP cluster
---

# Medik8s Certsuite Validation Skill

## Purpose

This skill provides a reusable workflow to validate certsuite exception documentation for any Medik8s operator against actual certsuite test results on an OpenShift cluster.

**Applicable Operators:**
- SNR (Self Node Remediation)
- FAR (Fence Agents Remediation)
- NHC (Node Health Check)
- NMO (Node Maintenance Operator)
- MDR (Machine Deletion Remediation)
- SBR (Storage-Based Remediation)

**Use Cases:**
- RHWA-1616: SNR certsuite validation
- RHWA-1617: FAR certsuite validation  
- RHWA-449 Epic: All Medik8s operators implementing telco best practices

## When to Use

Invoke this skill when:
- User has an OCP cluster and wants to validate certsuite exceptions documentation
- User says "validate certsuite exceptions" or "run certsuite validation"
- User mentions RHWA-449, RHWA-1616, RHWA-1617, or similar certsuite-related tickets
- User wants to verify operator compliance with Red Hat best practices

## Execution rules

**Collect all required inputs upfront in a single message before doing
anything else.** Once inputs are confirmed, run all phases automatically
with zero questions.

Make every decision unilaterally:
- Default `Operator Namespace` to `openshift-workload-availability` if not given
- If `Exceptions Doc Path` is not given but operator name is known, infer it
  as `docs/certsuite_exceptions.md` in the operator repo
- If a test result is SKIP (not FAIL), mark as "skipped" — no question needed
- If `claim.json` is not provided, derive expected test IDs from the
  exceptions doc section headers

**Surface warnings** during processing and in the final output:
- Print a one-line warning for each unexpected FAIL (not in exceptions doc)
- Collect all warnings into a **Warnings** block at the end of output

## Required Inputs

The skill requires these inputs (ask user if not provided):

1. **Operator Name**: `snr`, `far`, `nhc`, `nmo`, `mdr`, `sbr`
2. **Operator Namespace**: Default `openshift-workload-availability`
3. **Bundle Image**: e.g., `quay.io/medik8s/self-node-remediation-operator-bundle:latest`
4. **Exceptions Doc Path**: Path to `docs/certsuite_exceptions.md` in operator repo
5. **OCP Version**: e.g., `5.0`, `4.22` (for context)
6. **Test List**: Either:
   - Path to prior certsuite `claim.json` results file, OR
   - Path to operator's `docs/certsuite_exceptions.md` (will extract test IDs from section headers)

**IMPORTANT**: Do not assume which tests will fail. Extract the actual test IDs from user-provided input before starting validation.

## Prerequisites

### Tools Required

```bash
# Verify tools
oc version                    # OpenShift CLI
operator-sdk version          # Should be v1.34.1 or later
jq --version                  # JSON processor
yq --version                  # YAML processor (v4.x)
git --version                 # For repo cloning

# Update operator-sdk if needed (user indicated it should be available but verify latest)
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
curl -LO https://github.com/operator-framework/operator-sdk/releases/download/v1.34.1/operator-sdk_${OS}_${ARCH}
chmod +x operator-sdk_${OS}_${ARCH}
sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
```

### Cluster Requirements

- **Access**: Cluster-admin privileges
- **State**: Fully functional with ≥3 worker nodes
- **Network**: Outbound access to quay.io
- **OCP**: Version 4.15+ (5.0 recommended for RHWA-449)

### Repository Clones

```bash
# Clone operator repository
git clone https://github.com/medik8s/<operator-name>.git
cd <operator-name>

# Clone certsuite (if not already cloned)
git clone https://github.com/redhat-best-practices-for-k8s/certsuite.git
```

## Workflow

### Phase 0: Extract Test IDs

**From exceptions documentation:**

```bash
# Extract test IDs from docs/certsuite_exceptions.md section headers
# Pattern: ## N. [`test-id`](link)
grep -oP '(?<=## \d+\. \[\`)[^\`]+' docs/certsuite_exceptions.md > test-ids.txt

# Example output:
# access-control-cluster-role-bindings
# access-control-pod-host-path
# access-control-pod-host-pid
# ...
```

**From prior certsuite results:**

```bash
# Extract failed test IDs from claim.json
jq -r '.results[] | select(.state == "FAILED") | .testID.id' claim.json > test-ids.txt
```

**Create test validation checklist:**

```bash
# Generate numbered list for validation tracking
nl -v 1 -w 2 -s '. ' test-ids.txt
```

### Phase 1: Install Operator via OLM

**Why OLM?** Production-like installation that creates all resources certsuite tests, including OLM-managed RBAC.

```bash
# Set environment variables
export OPERATOR_NAMESPACE=openshift-workload-availability
export BUNDLE_IMG=<bundle-image-from-params>

# Navigate to operator repo
cd /path/to/<operator-repo>

# Install via OLM
make bundle-run OPERATOR_NAMESPACE=$OPERATOR_NAMESPACE BUNDLE_IMG=$BUNDLE_IMG

# Verify installation
oc -n $OPERATOR_NAMESPACE get csv
# Expected: PHASE = Succeeded

oc -n $OPERATOR_NAMESPACE get deployment
# Expected: All deployments Ready
```

**For operators requiring config CR to trigger DaemonSet:**
(e.g., SNR requires SelfNodeRemediationConfig)

```bash
# Apply operator-specific config CR
# See operator docs or examples/ directory
oc apply -f config/samples/<config-cr>.yaml

# Verify DaemonSet created (if applicable)
oc -n $OPERATOR_NAMESPACE get daemonset
```

### Phase 2: Install and Configure Certsuite

```bash
# Install certsuite binary
CERTSUITE_VERSION=v5.2.4  # Or latest from releases
curl -LO https://github.com/redhat-best-practices-for-k8s/certsuite/releases/download/${CERTSUITE_VERSION}/certsuite
chmod +x certsuite
sudo mv certsuite /usr/local/bin/

# Verify
certsuite version
```

**Create tnf_config.yml** (adapt for operator):

```yaml
---
targetNameSpaces:
  - name: <OPERATOR_NAMESPACE>

operators:
  - name: <operator-name>
    namespace: <OPERATOR_NAMESPACE>

targetPodLabels:
  # Adjust labels based on operator pod selectors
  - prefix: <OPERATOR_NAMESPACE>
    name: control-plane
    value: controller-manager
  - prefix: <OPERATOR_NAMESPACE>
    name: app.kubernetes.io/component
    value: agent  # Or daemon, worker, etc.

managedDeployments:
  - name: controller-manager  # Or operator's deployment name
    namespace: <OPERATOR_NAMESPACE>

managedDaemonSets:
  # Add if operator has DaemonSets
  - name: <daemonset-name>
    namespace: <OPERATOR_NAMESPACE>

targetCrdFilters:
  - namePattern: "<operator-crd-pattern>.*"  # e.g., "selfnoderemediation.*"

certifiedcontainerinfo:
  - name: controller
    repository: quay.io/medik8s/<operator-image>

checkOwners: true

# DO NOT add exceptionLists - we want to see all failures
```

### Phase 3: Run Certsuite

```bash
certsuite run \
  --config-file tnf_config.yml \
  --output-dir ./certsuite-results \
  --log-level debug
```

**Expected runtime**: 10-30 minutes

**Output files**:
- `claim.json` - JSON test results (use for validation)
- `results.html` - HTML report (human-readable)
- `certsuite.log` - Execution log
- `junit-*.xml` - JUnit XML

### Phase 4: Validate Documented Exceptions

**Create validation helper script**:

```bash
cat > validate_exception.sh <<'SCRIPT'
#!/bin/bash
# Usage: ./validate_exception.sh <test-id> [results-file]

TEST_ID=$1
RESULTS=${2:-./certsuite-results/claim.json}

if [ ! -f "$RESULTS" ]; then
  echo "ERROR: Results file not found: $RESULTS"
  exit 1
fi

echo "=== Test: $TEST_ID ==="
jq -r --arg test "$TEST_ID" '
  .results[] | select(.testID.id == $test) | 
  {state, message: .check_details}
' "$RESULTS"

if [ $? -ne 0 ]; then
  echo "WARNING: Test ID not found in results"
fi
SCRIPT

chmod +x validate_exception.sh
```

**For each documented exception from test-ids.txt:**

1. **Read the exception doc**: Check what the operator's `docs/certsuite_exceptions.md` claims
2. **Check test result**: `./validate_exception.sh <test-id>`
3. **Verify cluster state**: Run commands to verify actual configuration matches doc
4. **Record findings**:
   - Test state (FAILED/PASSED/SKIPPED)
   - Doc accuracy (ACCURATE/INACCURATE/MISLEADING)
   - Discrepancies

**Example validation** (access-control-cluster-role-bindings):

```bash
# 1. Check test failed
./validate_exception.sh "access-control-cluster-role-bindings"
# Expected: state = "FAILED"

# 2. Verify actual ClusterRoleBinding (check doc for name)
oc get clusterrolebinding <name-from-doc> -o yaml

# 3. Compare with documentation claims
# - Does the ClusterRoleBinding exist?
# - Does it bind the expected ClusterRole?
# - Do permissions match documentation?
```

### Phase 5: Generate Validation Report

**Report Template**: `<operator>-certsuite-validation-report.md`

```markdown
# <Operator> Certsuite Exception Validation Report

**Date**: YYYY-MM-DD
**Cluster**: OCP <version>
**Operator Version**: <version>
**Certsuite Version**: <version>
**Jira**: <ticket-number>

## Executive Summary

- **Total Documented Exceptions**: X (Y HARD, Z SOFT)
- **Validated as Accurate**: A/X
- **False Positives**: B (doc claims failure, test passes)
- **Documentation Inaccuracies**: C

## Validation Results

| Exception | Test ID | Expected | Actual | Accurate? | Notes |
|-----------|---------|----------|--------|-----------|-------|
| 1 | test-name | FAILED | FAILED | ✓ | Matches |
| 2 | test-name | FAILED | PASSED | ✗ | FALSE POSITIVE |
| ... | ... | ... | ... | ... | ... |

## Discrepancies Found

### False Positives
[Tests that passed but doc claims they fail]

### Inaccurate Justifications
[Where explanation doesn't match actual behavior]

### Recommendations

1. [Required documentation updates]
2. [Code changes needed (if any)]
3. [Clarifications for next certsuite run]

## Appendices

### A. Full Test Results
[Link to claim.json or summary]

### B. Cluster State
```
oc -n <namespace> get all -o yaml
```
```

### Phase 6: Cleanup

```bash
# Remove operator
make bundle-cleanup OPERATOR_NAMESPACE=$OPERATOR_NAMESPACE

# Verify namespace cleaned
oc -n $OPERATOR_NAMESPACE get all

# Optionally delete namespace
oc delete namespace $OPERATOR_NAMESPACE
```

## Common Exception Patterns (Reference)

Across Medik8s operators, these patterns appear frequently. Use as reference material when validating exceptions, not as assumed requirements.

### 1. ClusterRoleBindings (access-control-cluster-role-bindings)
- **Pattern**: Metrics authentication, webhook auth delegation
- **Validation**: Check actual ClusterRoleBinding names and permissions
- **SNR example**: `metrics-auth-rolebinding` → `metrics-auth-role`

### 2. HostPath Mounts (access-control-pod-host-path)
- **Pattern**: DaemonSets needing device access
- **Validation**: Check `volumes[].hostPath` in DaemonSet spec
- **SNR example**: `/dev` mount for watchdog device

### 3. HostPID (access-control-pod-host-pid)
- **Pattern**: DaemonSets needing host process access
- **Validation**: Check `spec.template.spec.hostPID`
- **SNR example**: `true` for nsenter-based reboot

### 4. kube-system RoleBindings (access-control-pod-role-bindings)
- **Pattern**: OLM-created for webhook operators
- **Validation**: Check `oc -n kube-system get rolebinding`
- **IMPORTANT**: Only exists in OLM installations! Verify by checking for CSV existence and ownerReferences on RoleBinding
- **SNR example**: `service-auth-reader`

### 5. Root UID / Privileged (access-control-security-context-non-root-user-id-check)
- **Pattern**: DaemonSets needing privileged operations
- **Validation**: Check `securityContext.runAsUser` and `privileged`
- **Common**: DaemonSet runs as root, controller-manager as non-root

### 6. Lifecycle Hooks (lifecycle-container-poststart, lifecycle-container-prestop)
- **Pattern**: Often SOFT exceptions (design choice, not requirement)
- **Validation**: Check `lifecycle.postStart` and `lifecycle.preStop`
- **Common**: DaemonSets omit hooks, Deployments include them

### 7. Owner Type (lifecycle-pod-owner-type)
- **Pattern**: DaemonSet for per-node coverage
- **Validation**: Check pod `ownerReferences[].kind`
- **Common**: Agent/daemon pods owned by DaemonSet

### 8. Tolerations (lifecycle-pod-toleration-bypass)
- **Pattern**: Control-plane, master, infra node tolerations
- **Validation**: Check `spec.template.spec.tolerations[]`
- **Common**: 3-4 tolerations for cluster-wide monitoring

### 9. Privileged SCC (operator-install-status-no-privileges)
- **Pattern**: CSV declares privileged SCC for DaemonSets
- **Validation**: Check pod annotation `openshift.io/scc`
- **Common**: DaemonSet = privileged, Deployment = restricted

## Troubleshooting

### OLM Installation Issues

**Issue**: CSV stuck in "Installing" phase
```bash
# Check OLM operator logs
oc -n openshift-operator-lifecycle-manager logs deployment/olm-operator

# Check catalog source
oc -n openshift-marketplace get catalogsource
```

**Issue**: DaemonSet pods in CrashLoopBackOff
```bash
# Check PSA labels on namespace
oc get ns $OPERATOR_NAMESPACE -o yaml | grep -A5 labels
# Should have: pod-security.kubernetes.io/enforce: privileged
```

### Certsuite Issues

**Issue**: No pods found by certsuite
```bash
# Verify label selectors in tnf_config.yml match actual pods
oc -n $OPERATOR_NAMESPACE get pods --show-labels
```

**Issue**: Test not in results
```bash
# Check certsuite log
grep <test-name> ./certsuite-results/certsuite.log
```

## Success Criteria

**Complete success**:
- All documented exceptions result in certsuite test state = FAILED
- All justifications match actual cluster state
- No undocumented failures

**Acceptable** (requires doc updates):
- ≥80% exceptions validated as accurate
- Minor clarifications needed (e.g., "OLM-only" notes)
- No major justification errors

**Requires investigation**:
- ≥3 false positives (test passes but doc claims failure)
- ≥2 major justification errors
- New undocumented exceptions found

## References

- **Certsuite Repository**: https://github.com/redhat-best-practices-for-k8s/certsuite
- **Test Catalog**: https://github.com/redhat-best-practices-for-k8s/certsuite/blob/main/CATALOG.md
- **Medik8s Operators**: https://github.com/medik8s/
- **RHWA-449 Epic**: Telco best practices implementation

## Related Skills

- `medik8s-github-report-status`: Track Medik8s GitHub issues/PRs
- `rpm-lockfile-update`: Update RPM lockfiles (downstream)

## Example Invocations

**SNR validation (RHWA-1616)**:
```
Validate SNR certsuite exceptions using:
- Operator: snr
- Namespace: openshift-workload-availability
- Bundle: quay.io/medik8s/self-node-remediation-operator-bundle:latest
- Doc: /tmp/claude-4203201/repos/self-node-remediation/docs/certsuite_exceptions.md
- OCP: 5.0
```

**FAR validation (RHWA-1617)**:
```
Validate FAR certsuite exceptions using:
- Operator: far
- Namespace: openshift-workload-availability  
- Bundle: quay.io/medik8s/fence-agents-remediation-operator-bundle:latest
- Doc: /path/to/fence-agents-remediation/docs/certsuite_exceptions.md
- OCP: 5.0
```

## Notes

- **Always use OLM installation** (`make bundle-run`) for production-like validation
- **Do not add exceptionLists** to tnf_config.yml - we want to see all failures
- **Document OLM-only exceptions** clearly (e.g., service-auth-reader RoleBinding)
- **Generate reports** even for successful validations (provides audit trail)
- **Archive results** with Jira ticket reference for future comparison
- **operator-sdk** should already be available; verify latest version before starting
