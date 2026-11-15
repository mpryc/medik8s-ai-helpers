---
name: medik8s-github-report-status
description: Use when asked for Medik8s open PR/MR status, PR dashboard, org-wide report, or hanging fruits across medik8s GitHub repos and dragonfly GitLab group.
---

# Medik8s PR/MR Status Report

Generate a comprehensive dashboard of all open PRs/MRs across:
- **GitHub:** `medik8s` organization (PRs)
- **GitHub:** `openshift/release` PRs touching `ci-operator/config/medik8s/` (OCP CI config)
- **GitLab:** `dragonfly` group at `gitlab.cee.redhat.com` (MRs)

## Execution rules

**No required inputs — run all phases automatically with zero questions.**
The user may pass optional flags (`--light`, `--security`); if no flags are
given, run the full report (Phases 1-5).

Make every decision unilaterally:
- If a GitHub API call rate-limits, wait 10 s and retry once
- If a GitLab API call fails (VPN issue), mark that group "unavailable" and
  continue with what is reachable
- If CodeRabbit review data is unavailable for a PR, mark CR column ⚪

**Surface warnings** during processing and in the final output:
- Print a one-line warning for any API failures or stale data
- Collect all warnings into a **Warnings** block at the end of the report

## Flags

| Flag | Effect |
|------|--------|
| *(none)* | Full report: Phases 1-5 (CI + metadata + Track A + Track B reviews + Slack draft) |
| `--light` | Light report: Phases 1-2 + 4-5 only (CI + metadata, **skip all code reviews**). Review and CR columns show ⚪ instead. Hanging Fruit classification uses only CI status + merge conflicts (ignores Review/CR). |
| `--security` | Full report + Track C security review in Phase 3 |

## Output Format

Two tables presented to the user:

1. **Hanging Fruits** — PRs ready to merge (both Review and CR are 🟢, no merge conflicts)
2. **All Other Open PRs** — everything else
3. **Draft Slack message** — ready-to-send review request for the user's own Hanging Fruit PRs

Both tables use identical columns:

```
# | Source | Repo | PR/MR | Title | Author | Draft | Updated | CI | +/- | Files | Review | CR | Me | Key Findings
```

When `--security` is passed, add a `Sec` column after `Me`:

```
# | Source | Repo | PR/MR | Title | Author | Draft | Updated | CI | +/- | Files | Review | CR | Me | Sec | Key Findings
```

**Me column color indicators:**
- 🟢 = I authored this PR/MR (author is `razo7` on GitHub or `oraz` on GitLab), OR I have a current review on it (no commits pushed after my latest review)
- 🟡 = I reviewed this PR/MR, but commits were pushed after my latest review (stale review — re-review needed)
- ⚪ = I did not author and have not reviewed this PR/MR

**Sec column color indicators:** 🟢 no security issues, 🟡 low-risk findings, 🔴 high-risk security issues

**Source column:** `GH` for GitHub medik8s PRs, `OCP` for openshift/release PRs, `GL` for GitLab MRs

**PR/MR column — clickable hyperlinks (mandatory):**
The PR/MR column MUST render as a markdown hyperlink: `[#NNN](url)` for GitHub PRs, `[!NNN](url)` for GitLab MRs.
URL patterns:
- GitHub medik8s: `[#NNN](https://github.com/medik8s/<repo>/pull/NNN)`
- openshift/release: `[#NNN](https://github.com/openshift/release/pull/NNN)`
- GitLab: `[!NNN](https://gitlab.cee.redhat.com/dragonfly/<repo>/-/merge_requests/NNN)`

Use the `url` or `web_url` field from Phase 1 API responses when available, to avoid constructing wrong URLs.

**Color indicators (no words):**
- CI column: 🟢 pass, 🟡 pending/partial, 🔴 fail, ⚪ none/N/A
- Review column: 🟢 green, 🟡 yellow, 🔴 red
- CR column: 🟢 green, 🟡 yellow, 🔴 red

**Sort order:** Updated (oldest first) → Repo name (alpha) → PR number (ascending)

**Legend** at the bottom: `🟢 pass/green 🟡 pending/partial/yellow 🔴 fail/red ⚪ none/N/A`

After the tables, show a **Summary** metrics table:
```
Total open PRs | Both 🟢 | Any 🟡 | Any 🔴 | CI passing | CI failing | CI pending/none | Drafts | Merge conflicts | Stale (>6 months)
```

When `--security` is passed, add to the summary:
```
Sec 🟢 | Sec 🟡 | Sec 🔴 | Attack surface: increased | decreased | unchanged
```

## Workflow

### Phase 1a — Fetch all open GitHub PRs

Run:
```bash
gh search prs --owner medik8s --state open --limit 100 \
  --json repository,number,title,author,createdAt,updatedAt,url,isDraft
```

**Exclude** these repos (legacy/not relevant):
- `poison-pill` — archived, superseded by `self-node-remediation`
- `customized-user-remediation` — not actively maintained

### Phase 1b — Fetch openshift/release PRs touching medik8s CI config

Two-step process to find PRs that impact medik8s OCP CI:

**Step 1** — Search for candidate PRs:
```bash
gh search prs --repo openshift/release --state open --limit 50 \
  --json number,title,author,updatedAt,url,isDraft -- "medik8s"
```

**Step 2** — Filter to only PRs that actually touch medik8s CI config files:
```bash
gh pr diff <number> --repo openshift/release --name-only | grep "medik8s"
```

Only include PRs where the diff touches paths under:
- `ci-operator/config/medik8s/`
- `ci-operator/jobs/medik8s/`
- `ci-operator/step-registry/medik8s/`

Discard PRs that mention "medik8s" only in title/description but don't touch these paths.

### Phase 1c — Fetch all open GitLab MRs

Run:
```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/groups/dragonfly/merge_requests?state=opened&per_page=100"
```

Key fields from the response:
- `references.full` — e.g. `dragonfly/rhwa-fbc!322` (extract repo + MR number)
- `title`, `author.username`, `draft`, `updated_at`, `web_url`
- `has_conflicts`, `detailed_merge_status` (mergeable/conflict/unchecked)
- `labels` — check for `konflux-nudge` (automated Renovate MRs)

For CI/pipeline status per MR, fetch:
```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/<project_id>/merge_requests/<iid>/pipelines"
```

Map `detailed_merge_status`: `mergeable` → 🟢, `unchecked` → 🟡, `conflict` → 🔴

### Phase 2 — Fetch CI status and metadata

**GitHub PRs** — Launch **parallel Explore agents** in batches of ~10 PRs each. Each agent runs:

```bash
gh pr checks <number> --repo medik8s/<repo>
gh pr view <number> --repo medik8s/<repo> \
  --json statusCheckRollup,reviewDecision,reviews,latestReviews,labels,additions,deletions,changedFiles,commits,author
```

**GitLab MRs** — For each MR, fetch pipeline, diff stats, and approvals via REST API:

```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/<project_id>/merge_requests/<iid>/pipelines"
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/<project_id>/merge_requests/<iid>/changes?access_raw_diffs=true"
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/<project_id>/merge_requests/<iid>/approvals"
```

Each agent reports per PR/MR:
- CI status mapped to color (🟢/🟡/🔴/⚪)
- Review decision (APPROVED, CHANGES_REQUESTED, REVIEW_REQUIRED)
- Additions, deletions, changed files count
- Failing check names (if any)
- **Me column data:**
  - GitHub: check if `author.login == "razo7"` → 🟢 (authored). Otherwise check `latestReviews` for a review by `razo7`. If found, compare its `submittedAt` against the latest `commits[].committedDate` — if review is newer → 🟢, if commit is newer → 🟡 (stale). If no review → ⚪.
  - GitLab: check if MR author is `oraz` → 🟢 (authored). Otherwise check `approvals.approved_by` for `oraz`. If approved → 🟢. If not approved → ⚪. (GitLab approvals are automatically revoked on new pushes, so no stale check needed.)

### Phase 3 — Code reviews (two or three parallel tracks)

**Skip this entire phase when `--light` is passed.** Set Review and CR columns to ⚪ for all PRs/MRs.

Launch Track A and Track B simultaneously. If `--security` flag is passed, also launch Track C in parallel.

**Track A — Manual review agents** (general-purpose, batches of ~10):
- Fetch `gh pr diff <number> --repo medik8s/<repo>`
- Assess: correctness bugs, breaking changes, test coverage gaps, API compatibility, error handling, resource leaks
- Assign verdict: 🟢 (no issues, ready for LGTM) / 🟡 (minor issues) / 🔴 (critical issues)
- Return: `REPO#NUMBER | VERDICT | ONE-LINE-FINDING`

**Track B — CodeRabbit review agents** (`coderabbit:code-reviewer` subagent type, batches of ~10):
- Same diff, independent local assessment
- **Local only — NEVER post comments to GitHub**
- Same verdict scale and return format

**Track C — Security review agents** (general-purpose, batches of ~10) — **only when `--security` is passed**:
- Same diff, security-focused differential review
- Assess across 7 dimensions:
  1. RBAC changes — new permissions, service account changes
  2. Container security — securityContext, privileges, capabilities, readOnlyRootFilesystem
  3. Input validation — injection risks in shell commands, API calls, unvalidated config
  4. Secrets handling — credentials in code, env vars, sensitive data in logs
  5. Supply chain — dependency changes, image refs, action version pins (SHA vs tag)
  6. Race conditions — concurrent shared resource access without locking
  7. Info disclosure — sensitive data in logs, error messages, status fields
- Classify findings with CWE where applicable
- Assess attack surface change: increased / decreased / unchanged
- Assign verdict: 🟢 (no security issues) / 🟡 (low-risk findings) / 🔴 (high-risk security issues)
- Return: `REPO#NUMBER | VERDICT | ATTACK_SURFACE | ONE-LINE-FINDING`
- **Local only — NEVER post comments to GitHub or GitLab**

### Phase 4 — Compile and present

1. Merge all agent results into a single dataset
2. Classify each PR:
   - **Hanging Fruit** if: both Review and CR are 🟢, AND no merge conflict (when `--security`: Sec must also be 🟢). When `--light`: CI is 🟢 AND GitHub reviewDecision is APPROVED AND no merge conflict (Review/CR columns show ⚪ but GitHub's own review status is still used for classification).
   - **All Other** otherwise
3. For Hanging Fruits, add a `What's Blocking` column instead of `Key Findings` showing what's needed to merge (e.g., "Undraft + `/lgtm`", "`/approve` + `/lgtm`")
4. Sort both tables: Updated (oldest first) → Repo (alpha) → PR# (asc)
5. Number rows sequentially across both tables (Hanging Fruits first, then continue numbering)
6. Render the PR/MR column as markdown hyperlinks (e.g., `[#14](https://github.com/medik8s/.github/pull/14)`, `[!322](https://gitlab.cee.redhat.com/dragonfly/rhwa-fbc/-/merge_requests/322)`) — use the `url`/`web_url` from Phase 1 data
7. Present tables and summary to user

### Phase 5 — Draft Slack review request

After presenting the tables, draft a Slack message the user can send to their team asking for reviews on their own Hanging Fruit PRs.

**Categorize the user's Hanging Fruit PRs into two groups:**

1. **Waiting for review** — CI is 🟢 (or 🟡 pending). These go into the Slack message.
2. **Needs CI fix first** — CI is 🔴. These are listed separately below the Slack draft as a note to the user. Do NOT include them in the Slack message (no point asking for review when CI is broken).

**Template (only "Waiting for review" PRs/MRs):**
```
Hey team! :wave:
I have a few PRs/MRs that are reviewed and CI green — would appreciate an approval on them:

*GitHub — medik8s:*
• `<repo>` — <#NNN: title> (<url>)
• ...

*GitHub — openshift/release (OCP CI):*
• <#NNN: title> (<url>)
• ...

*GitLab — dragonfly:*
• `<repo>` — <!NNN: title> (<url>)
• ...

All CI green and approved — would appreciate a merge! Thanks! :pray:
```

Vary the closing line by mode:
- Default: `All have been reviewed (manual + CodeRabbit) with no issues found. Thanks! :pray:`
- `--security`: `All have been reviewed (manual + CodeRabbit + security) with no issues found. Thanks! :pray:`
- `--light`: `All CI green and approved — would appreciate a merge! Thanks! :pray:`

```
```

If there are no user PRs/MRs on a platform, omit that section entirely (don't show an empty header).
For `openshift/release` PRs, no repo sub-name is needed — just the PR number and title.

**Rules:**
- Only include PRs authored by the user (razo7) from the Hanging Fruits table
- All razo7 PRs to Medik8s are auto-approved — do NOT mention approval status or categorize by it
- List all reviewable PRs in a single flat list (no sub-categories like "already approved" vs "will undraft")
- Each bullet: repo name, linked PR number+title
- If a PR has a `do-not-merge/hold` label, include `/unhold` in the blocking info
- If CI is 🔴, exclude from the Slack message and list separately as "Needs CI fix first"
- Present the draft to the user — do NOT send it automatically
- User decides whether and where to post it

## Critical Rules

1. **Never post anything to GitHub or GitLab** — all reviews are local, in-session only
2. **Always exclude `poison-pill` and `customized-user-remediation`** — legacy/not relevant
3. **Always use color indicators** — never use words like PASS/FAIL/GREEN/YELLOW in the CI/Review/CR columns
4. **Hanging Fruits table always comes first** — these are the actionable items
5. **Review tracks depend on flags** — `--light` skips Phase 3 entirely (Review/CR columns show ⚪); default runs Track A + B; `--security` adds Track C
6. **Batch agents at ~10 PRs each** — balances parallelism with context quality
7. **Never auto-send the Slack draft** — always present it for the user to review and send manually
8. **Check for `do-not-merge/hold` labels** — if present, include `/unhold` in the blocking info
9. **razo7 PRs are auto-approved** — never categorize by approval status; all go in one flat "waiting for review" list
10. **CI 🔴 PRs/MRs skip the Slack draft** — don't ask the team to review PRs/MRs with broken CI; list them separately as "Needs CI fix first"
11. **GitLab uses `$GITLAB_PRIVATE_TOKEN`** — always use the REST API with this env var; WebFetch is unreliable for GitLab
12. **Slack message separates all three sources** — `GitHub — medik8s`, `GitHub — openshift/release (OCP CI)`, `GitLab — dragonfly`; omit any section with no user PRs/MRs
13. **openshift/release PRs must touch medik8s paths** — search by keyword, then filter by `gh pr diff --name-only` for paths under `ci-operator/config/medik8s/`, `ci-operator/jobs/medik8s/`, or `ci-operator/step-registry/medik8s/`
13. **GitLab MR notation uses `!`** — e.g. `!322`, not `#322`
