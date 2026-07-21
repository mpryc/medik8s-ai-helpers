---
description: Run upstream community release (tag, build & push images, create community catalog PRs)
argument-hint: <operator> --version <version> --previous <prev> --ocp <ocp_version> [--skip-range-lower <ver>] [--dry-run]
---

## Name
medik8s-release:upstream

## Synopsis
```
/medik8s-release:upstream <operator> --version <version> --previous <prev> --ocp <ocp_version> [--skip-range-lower <ver>] [--dry-run]
```

## Description
The `medik8s-release:upstream` command runs the upstream community release for a medik8s operator. This includes creating a git tag, triggering the GitHub Actions workflow to build and push images, and creating community catalog PRs for K8s OperatorHub and OKD/OCP.

There are two paths: using the **community release script** (for batch releases of multiple operators) or the **manual workflow** (for a single operator). This command implements the manual workflow approach.

### What Gets Created
- Git tag `v<version>` on the upstream repo
- Container images on `quay.io/medik8s/`:
  - `<operator>:v<version>` (operator)
  - `<operator>-bundle:v<version>` (OLM bundle)
  - `<operator>-catalog:v<version>` (catalog index)
- GitHub Release (draft) with `bundle.tar.gz` artifact
- Community catalog PRs on forks, then upstream PRs

### Operator-Specific Notes
- **NHC**: requires `--skip-range-lower` parameter; has companion repos (node-remediation-console, must-gather) that also need tagging
- **MDR**: no K8s community release (not supported on vanilla K8s)
- **NMO**: version scheme is 5.x (e.g., 5.7.0), not 0.x

## Implementation

### Phase 1: Validate Inputs
1. Resolve operator short name to full repo name
2. Verify the version tag doesn't already exist:
   ```bash
   gh api repos/medik8s/${repo}/git/refs/tags/v${version} 2>/dev/null
   ```
3. Verify the release branch or commit exists
4. For NHC, ensure `--skip-range-lower` is provided

### Phase 2: Create Tag
Discover the upstream repo clone locally. **Do NOT assume any fixed directory layout.**
```bash
# Discover upstream clone by repo name + GitHub remote
UPSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${repo}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -q "github.com/medik8s/${repo}" && echo "$dir" && break
done)
# If not found, ask the user for the path or use gh CLI to create the tag remotely

cd "$UPSTREAM_DIR"
git fetch origin
git tag -s v${version} -m "Release v${version}"
git push origin v${version}
```

For NHC companions:
```bash
# Tag node-remediation-console and must-gather at their current main HEAD
for companion in node-remediation-console must-gather; do
    # Discover each companion repo the same way
    COMPANION_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${companion}" 2>/dev/null | while read dir; do
      git -C "$dir" remote -v 2>/dev/null | grep -q "github.com/medik8s/${companion}" && echo "$dir" && break
    done)
    cd "$COMPANION_DIR"
    git fetch origin
    git tag -s v${version} -m "Release v${version}"
    git push origin v${version}
done
```

### Phase 3: Build & Push Images
Trigger the GitHub Actions release workflow:
```bash
gh workflow run release.yml \
    --repo medik8s/${repo} \
    --ref v${version} \
    -f operation=build_and_push_images \
    -f version=${version} \
    -f previous_version=${previous} \
    -f checkout_ref=v${version}
```

For NHC, add: `-f skip_range_lower=${skip_range_lower}`
For FAR/NHC, add: `-f ocp_version=${ocp_version}`

Wait for workflow completion:
```bash
# Poll until done
gh run list --repo medik8s/${repo} --workflow release.yml --limit 1 --json status,conclusion
```

### Phase 4: Verify Images
```bash
for suffix in "" "-bundle" "-catalog"; do
    skopeo inspect "docker://quay.io/medik8s/${repo}${suffix}:v${version}" > /dev/null 2>&1 && echo "OK: ${repo}${suffix}:v${version}" || echo "MISSING: ${repo}${suffix}:v${version}"
done
```

### Phase 5: Create Community PRs
```bash
# K8s community (skip for MDR)
gh workflow run release.yml \
    --repo medik8s/${repo} \
    --ref v${version} \
    -f operation=create_k8s_release_pr \
    -f version=${version} \
    -f previous_version=${previous}

# OKD community
gh workflow run release.yml \
    --repo medik8s/${repo} \
    --ref v${version} \
    -f operation=create_okd_release_pr \
    -f version=${version} \
    -f previous_version=${previous} \
    -f ocp_version=${ocp_version}
```

### Phase 6: Report
Print summary with:
- Tag URL
- GitHub Release URL
- Image tags on quay.io
- Community PR branch names and links

## Return Value
Summary of all created artifacts: tag, images, GitHub Release, community PR branches.

## Examples

1. **Release SNR 0.13.0**:
   ```
   /medik8s-release:upstream SNR --version 0.13.0 --previous 0.12.1 --ocp 4.22
   ```

2. **Release NHC 0.12.0 with skipRange**:
   ```
   /medik8s-release:upstream NHC --version 0.12.0 --previous 0.11.0 --ocp 4.22 --skip-range-lower 0.10.0
   ```

3. **Dry-run for FAR**:
   ```
   /medik8s-release:upstream FAR --version 0.8.0 --previous 0.7.0 --ocp 4.22 --dry-run
   ```

## Arguments
- `$1`: Operator short name (SNR, NHC, FAR, MDR, NMO, SBR)
- `--version <version>`: Release version (without `v` prefix)
- `--previous <prev>`: Previous version for OLM `replaces` field
- `--ocp <ocp_version>`: Target OpenShift version (e.g., 4.22)
- `--skip-range-lower <ver>`: Lower bound for OLM `skipRange` (required for NHC)
- `--dry-run`: Show what would be done without executing

## See Also
- `/medik8s-release:status` — Check release readiness
- `/medik8s-release:downstream` — Prepare midstream after upstream release
- `/medik8s-release:verify-bundle` — Validate the generated bundle
