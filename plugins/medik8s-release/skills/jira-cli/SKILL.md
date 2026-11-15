---
name: jira-cli
description: Use for interacting with Jira (redhat.atlassian.net). Create, view, edit, comment on, and search issues using the jira CLI tool. Handles Jira Cloud ADF formatting for complex content.
reference: https://github.com/ankitpokhrel/jira-cli
---

# Jira CLI

Interact with [redhat.atlassian.net](https://redhat.atlassian.net) (Jira Cloud) using the [jira-cli](https://github.com/ankitpokhrel/jira-cli) tool.

## Prerequisites

The `jira` command must be installed and initialized by the user via `jira init` before use. Do not run `jira init` — it requires interactive input.

The CLI operates on a single project configured during `jira init`. To work with multiple projects, separate configuration files are needed.

Config: `~/.config/.jira/.config.yml` — set `installation: Cloud`, `server: https://redhat.atlassian.net`, `auth_type: basic`.

## Output Modes

- Use `--raw` with `list` and `view` commands to get JSON output instead of formatted tables
- Use `--no-input` with `edit` and `create` commands to avoid interactive prompts

## Commands

### View and Search

```bash
jira issue list
jira issue list -s "To Do"
jira issue list --raw
jira issue list -q 'project=RHWA AND status="To Do"'
jira issue view ISSUE-1
jira issue view ISSUE-1 --raw
```

### Create

```bash
jira issue create -t "Task" -s "Summary" -b 'Description' --no-input
jira issue create -t "Sub-Task" -P ISSUE-1 -s "Summary" -b 'Description' -C component1,component2 --no-input
```

### Edit

```bash
jira issue edit ISSUE-1 -s "New summary" -b "New description" --no-input
jira issue edit ISSUE-1 --label p1 --no-input
jira issue edit ISSUE-1 --label p2 --component comp1 --component comp2 --no-input
```

Remove labels or components by appending `-`:

```bash
jira issue edit ISSUE-1 --label -p1 --no-input
jira issue edit ISSUE-1 --component -comp1 --no-input
```

### Assign

```bash
jira issue assign ISSUE-1 "Jon Doe"
jira issue assign ISSUE-1 x               # unassign
```

### Comment

Pipe body via stdin to avoid hanging:

```bash
echo "Comment body" | jira issue comment add ISSUE-1 --no-input
```

Do NOT pass the comment body as a positional argument — the CLI may hang waiting for interactive input.

## Jira Cloud ADF Formatting

**CRITICAL: The jira CLI over-escapes wiki markup.** Its `md.ToJiraMD()` round-trip adds spurious backslash escapes: `-` → `\-`, `()` → `\(\)`, `#` numbered lists → `h1.` headings, `RHWA-449` → `RHWA\-449`. This affects both `-b` flag and piped `jira issue edit` input.

**Always use the REST API v3 with ADF for any formatted description.** The jira CLI is only safe for plain-text single-line descriptions.

### What works with the CLI

- Simple single-line descriptions (`-b "plain text"`)
- Creating issues with placeholder description (then update via REST API)
- Labels, components, assignments, fix versions

### What breaks with the CLI (do NOT use for these)

- Multi-line descriptions — `\n` in `-b` is literal, not a line break
- Bullet/numbered lists — `#` becomes `h1.` headings
- Hyphens get backslash-escaped — `RHWA-449` → `RHWA\-449`
- Parentheses get backslash-escaped — `(CRITICAL)` → `\(CRITICAL\)`
- Links inside table cells — wiki markup `[text|url]` gets split across columns
- Piped wiki markup via `jira issue edit` — still over-escapes
- Wiki-style links `[text|url]` in bullets — URL gets duplicated: renders as `[text| url url]`
- Bare URLs in bullets — URL gets duplicated in the ADF output
- `h3.` headings work correctly, but paragraph text before `*` bullets merges the first bullet into the paragraph

### When to use the REST API directly

For ANY description with formatting (headings, lists, links, tables), bypass the CLI and use the Jira REST API v3 with ADF JSON directly.

**CRITICAL: Use `update` syntax, NOT `fields` syntax.** The RHWA project (and possibly others) has screen security that blocks `{"fields": {"description": ...}}` with error "Field 'description' cannot be set. It is not on the appropriate screen." The `update` syntax bypasses this:

```bash
# CORRECT — update syntax (bypasses screen security)
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -X PUT \
  "https://redhat.atlassian.net/rest/api/3/issue/ISSUE-KEY" \
  -H "Content-Type: application/json" \
  -d '{"update": {"description": [{"set": <adf-doc>}]}}'

# WRONG — fields syntax (blocked by screen security on RHWA)
# curl ... -d '{"fields": {"description": <adf-doc>}}'
```

ADF helper functions (Python):

```python
import json, urllib.request, base64, os

email = os.environ["JIRA_EMAIL"]
token = os.environ["JIRA_API_TOKEN"]
credentials = base64.b64encode(f"{email}:{token}".encode()).decode()

def text(t, marks=None):
    node = {"type": "text", "text": t}
    if marks:
        node["marks"] = marks
    return node

def link_text(label, url):
    return text(label, [{"type": "link", "attrs": {"href": url}}])

def para(*nodes):
    return {"type": "paragraph", "content": list(nodes)}

def heading(level, t):
    return {"type": "heading", "attrs": {"level": level}, "content": [text(t)]}

def bold(t):
    return text(t, [{"type": "strong"}])

def code(t):
    return text(t, [{"type": "code"}])

def bullet_list(items):
    return {"type": "bulletList", "content": [
        {"type": "listItem", "content": [para(*i) if isinstance(i, list) else para(i)]}
        for i in items
    ]}

def ordered_list(items):
    return {"type": "orderedList", "content": [
        {"type": "listItem", "content": [para(*i) if isinstance(i, list) else para(i)]}
        for i in items
    ]}

def th(t):
    return {"type": "tableHeader", "attrs": {}, "content": [
        para(text(t, [{"type": "strong"}]))
    ]}

def td(*nodes):
    return {"type": "tableCell", "attrs": {}, "content": [para(*nodes)]}

def trow(*cells):
    return {"type": "tableRow", "content": list(cells)}

def table(*rows):
    return {"type": "table", "attrs": {"isNumberColumnEnabled": False, "layout": "wide"}, "content": list(rows)}

doc = {"version": 1, "type": "doc", "content": [
    heading(3, "Title"),
    para(text("Text with "), link_text("a link", "https://example.com")),
    table(
        trow(th("Name"), th("Status")),
        trow(td(link_text("Repo", "https://github.com/org/repo")), td(text("OK"))),
    ),
]}

# Use update syntax, NOT fields
data = json.dumps({"update": {"description": [{"set": doc}]}}).encode()
req = urllib.request.Request(
    "https://redhat.atlassian.net/rest/api/3/issue/ISSUE-KEY",
    data=data, method="PUT",
    headers={"Authorization": f"Basic {credentials}", "Content-Type": "application/json"},
)
resp = urllib.request.urlopen(req)
```

### ADF gotchas

- **Use `update` syntax for descriptions** — `{"update": {"description": [{"set": <adf>}]}}` bypasses screen security; `{"fields": {"description": <adf>}}` is blocked on RHWA and possibly other projects
- Table `layout` values: `default`, `center`, `wide` — NOT `wide-breakout` (returns 400)
- `panel` type uses `panelType` attr: `info`, `note`, `warning`, `success`, `error`
- Auth: `curl -u "$JIRA_EMAIL:$JIRA_API_TOKEN"` — do NOT manually base64-encode; the `-u` flag handles it correctly
- Endpoint: `PUT /rest/api/3/issue/{key}` for description, `POST /rest/api/3/issue/{key}/comment` for comments
- Comment body uses the same ADF format in `{"body": <adf-doc>}`
- Always verify rendering with `jira issue view ISSUE-KEY --plain` after setting descriptions

### Recommended workflow for creating issues with formatted descriptions

1. Create the issue with `jira issue create` using a placeholder description
2. Immediately update the description via REST API with proper ADF JSON using `update` syntax
3. Verify rendering with `jira issue view ISSUE-KEY --plain`
4. Then set fix versions, links, etc. via `jira issue edit` or REST API

```bash
# Step 1: Create with placeholder
jira issue create -t "Task" -s "Summary" -b "placeholder" -C component --no-input

# Step 2: Set proper ADF description via update syntax
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -X PUT \
  "https://redhat.atlassian.net/rest/api/3/issue/ISSUE-KEY" \
  -H "Content-Type: application/json" \
  -d '{"update": {"description": [{"set": <adf-doc>}]}}'

# Step 3: Verify
jira issue view ISSUE-KEY --plain
```

## When to Use

- Creating, viewing, editing, or commenting on Jira issues
- Searching for issues with JQL queries
- Managing labels, components, and assignments
- Any interaction with redhat.atlassian.net that would otherwise require the web UI

## When NOT to Use

- Do not run `jira init` — the user must configure this interactively
- Do not use for Jira instances other than redhat.atlassian.net unless the user has configured them
