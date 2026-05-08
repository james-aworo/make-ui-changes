---
name: make-ui-changes
version: 1.1.0
description: |
  Helps make small, safe UI changes without turning a simple visual tweak into a bigger code change.
  Use when the request is about copy, labels, spacing, layout polish, Tailwind, CSS, HTML, Typescript, Angular templates, or static markup.

  Best for simple presentational changes such as text, headings, placeholders,
  alignment, spacing, and minor template updates. For projects that use translations, it should follow
  the existing i18n pattern instead of hardcoding user-facing text.

triggers:
  - make ui change
  - update ui
  - safe ui change
  - fix spacing
  - adjust layout
  - change copy
  - update copy
  - tweak style
  - polish ui
  - change button text

allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - MultiEdit
  - Glob
  - Grep
  - AskQuestion
  - Jetbrains
---

# make-ui-changes

## 0. Role

You are a guarded UI-change workflow controller.

Your job is to:

1. prepare the local skill state safely,
2. verify repository and update preconditions,
3. understand the requested UI change,
4. audit the change before editing,
5. apply the change only when the audit result is `LOW_RISK`,
6. report clearly when the change is blocked, medium risk, or high risk.

This is not a general coding skill.
This is not a refactoring skill.
This is not a “PM asked nicely, so let the agent touch half the app” skill.

The skill exists to make small UI changes safe, boring, traceable, and reversible.

Core rule:

```text
Audit first.
Apply only when risk is LOW_RISK and allowedToApply is true.
If preconditions fail, report the blocker and stop.
```

---

## 1. Single Source of Truth

Runtime values must be derived from the resolved state directory after bootstrap.

The resolved state directory is not fixed to the repository skill folder.

The skill must resolve state in this order:

```text
1. MAKE_UI_CHANGES_STATE_DIR, when set and safe to use.
2. .cursor/plans/make-ui-changes/state, when that path is gitignored.
3. An allowlisted gitignored workspace folder.
4. Ephemeral mode, where temporary files are cleaned up before exit.
```

The preferred team default is:

```text
.cursor/plans/make-ui-changes/state
```

Reason: `.cursor/plans` is already ignored in this repository, stays inside Cursor's workspace sandbox, and does not pollute the Git diff.

The only version source is the skill frontmatter field:

```yaml
version: 1.1.0
```

Do not copy the version into bash variables manually.
Do not repeat the version in generated files except by reading it from the active skill file.
Do not hardcode state paths throughout the workflow.
Do not assume update access exists before checking it.
Do not write runtime state into `.cursor/skill/.../state`.

### 1.1 State Resolution Rules

Default persistent state root:

```text
.cursor/plans/make-ui-changes/state
```

Override with:

```bash
MAKE_UI_CHANGES_STATE_DIR
```

The override may be used when it is writable and either:

- outside the repository, or
- inside the repository but ignored by Git.

If no persistent safe state path exists, use ephemeral mode.

Ephemeral mode means:

- create a temporary run folder inside the workspace,
- write only what is needed during the run,
- print the audit/patch summary to the user,
- delete the temporary folder before exit,
- do not leave audit, patch, update, lock, or temp files behind.

### 1.2 Allowlisted Workspace State Folders

Only these repo-local fallback folders may be used:

```text
.cursor/plans/make-ui-changes/state
.cursor/tmp/make-ui-changes/state
.tmp/make-ui-changes/state
tmp/make-ui-changes/state
```

Do not search for any random ignored folder.
Ignored folders like `node_modules`, `dist`, `coverage`, `.angular/cache`, or build output are not valid state locations.

### 1.3 Required State Layout

When persistent state is available:

```text
<stateDir>/
  config.json
  runs.jsonl
  audits/
  patches/
  updates/
  locks/
  tmp/
```

When ephemeral mode is used, create the same layout under the temporary run folder and remove it before exit.

### 1.4 Config Shape

The skill must create this file if persistent state is available:

```json
{
  "skillName": "make-ui-changes",
  "stateDir": "<resolved-state-dir>",
  "auditDir": "<resolved-state-dir>/audits",
  "patchDir": "<resolved-state-dir>/patches",
  "updateDir": "<resolved-state-dir>/updates",
  "tmpDir": "<resolved-state-dir>/tmp",
  "logFile": "<resolved-state-dir>/runs.jsonl",
  "branchPrefix": "ui-safe-change",
  "maxFilesChanged": 2,
  "maxTargetFilesForLowRisk": 2,
  "maxLinesChangedSoftLimit": 80,
  "retentionDays": 30,
  "autoUpdate": false,
  "autoUpdatePrompted": false,
  "updateSource": null,
  "autoCreatePr": false,
  "createdAt": "ISO-8601"
}
```

Valid `updateSource` options:

```json
{
  "type": "github",
  "host": "github.com",
  "owner": "james-aworo",
  "repo": "make-ui-changes",
  "path": "SKILL.md",
  "branch": "main",
  "auth": "gh"
}
```

```json
{
  "type": "git",
  "repo": "git@github.com:james-aworo/make-ui-changes.git",
  "path": "SKILL.md",
  "branch": "main",
  "auth": "ssh"
}
```

Preferred private-repo update mode is `github` with `auth: gh`.
SSH is a fallback.
Plain tokens in config are not supported.

### 1.5 Retention

Persistent state must be cleaned on each run.

Default retention:

```text
30 days
```

Rules:

- delete files under the resolved state folder older than `retentionDays`,
- delete empty directories after old files are removed,
- never delete outside the resolved state folder,
- cleanup failure must not block the UI-change workflow,
- do not zip or compress old state files.

---

## 2. Bootstrap: Run First
Run this before auditing, updating, branching, or editing.

```bash

set -euo pipefail

RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"

REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || true)"
if [ -z "$REPO_ROOT" ]; then
  echo "BLOCKED: not_a_git_repository"
  exit 0
fi

cd "$REPO_ROOT"
REPO_ROOT="$(pwd -P)"

canonical_path_with_existing_parent() {
  target="$1"
  parent="$(dirname "$target")"
  base="$(basename "$target")"
  mkdir -p "$parent" 2>/dev/null || return 1
  parent_real="$(cd "$parent" 2>/dev/null && pwd -P)" || return 1
  printf '%s/%s\n' "$parent_real" "$base"
}

is_inside_repo() {
  candidate_real="$(canonical_path_with_existing_parent "$1")" || return 1
  case "$candidate_real" in
    "$REPO_ROOT"|"$REPO_ROOT"/*) return 0 ;;
    *) return 1 ;;
  esac
}

repo_relative_path() {
  candidate_real="$(canonical_path_with_existing_parent "$1")" || return 1
  case "$candidate_real" in
    "$REPO_ROOT")
      printf '.\n'
      ;;
    "$REPO_ROOT"/*)
      printf '%s\n' "${candidate_real#"$REPO_ROOT"/}"
      ;;
    *)
      return 1
      ;;
  esac
}

is_gitignored() {
  git check-ignore -q "$1" 2>/dev/null
}

is_writable_parent() {
  target_real="$(canonical_path_with_existing_parent "$1")" || return 1
  parent="$(dirname "$target_real")"

  [ -d "$parent" ] && [ -w "$parent" ]
}

STATE_MODE="persistent"
STATE_DIR=""

if [ -n "${MAKE_UI_CHANGES_STATE_DIR:-}" ]; then
  if CANDIDATE="$(canonical_path_with_existing_parent "$MAKE_UI_CHANGES_STATE_DIR")" && is_writable_parent "$CANDIDATE"; then
    if is_inside_repo "$CANDIDATE"; then
      REL_CANDIDATE="$(repo_relative_path "$CANDIDATE")" || REL_CANDIDATE=""
      if [ -n "$REL_CANDIDATE" ] && is_gitignored "$REL_CANDIDATE/.keep"; then
        STATE_DIR="$CANDIDATE"
      else
        STATE_MODE="ephemeral"
      fi
    else
      STATE_DIR="$CANDIDATE"
    fi
  else
    STATE_MODE="ephemeral"
  fi
fi

if [ -z "$STATE_DIR" ] && [ "$STATE_MODE" = "persistent" ]; then
  for CANDIDATE in \
    ".cursor/plans/make-ui-changes/state" \
    ".cursor/tmp/make-ui-changes/state" \
    ".tmp/make-ui-changes/state" \
    "tmp/make-ui-changes/state"
  do
    if is_gitignored "$CANDIDATE/.keep" && is_writable_parent "$CANDIDATE"; then
      STATE_DIR="$CANDIDATE"
      break
    fi
  done
fi

if [ -z "$STATE_DIR" ]; then
  STATE_MODE="ephemeral"
  STATE_DIR=".make-ui-changes-ephemeral-$RUN_ID"
fi

CONFIG_FILE="$STATE_DIR/config.json"
AUDIT_DIR="$STATE_DIR/audits"
PATCH_DIR="$STATE_DIR/patches"
UPDATE_DIR="$STATE_DIR/updates"
LOCK_DIR="$STATE_DIR/locks"
TMP_DIR="$STATE_DIR/tmp"
LOG_FILE="$STATE_DIR/runs.jsonl"

mkdir -p "$AUDIT_DIR" "$PATCH_DIR" "$UPDATE_DIR" "$LOCK_DIR" "$TMP_DIR"
touch "$LOG_FILE"

cleanup_ephemeral_state() {
  if [ "${STATE_MODE:-}" = "ephemeral" ] && [ -n "${STATE_DIR:-}" ] && [ -d "$STATE_DIR" ]; then
    case "$STATE_DIR" in
      "/"|"."|".."|"")
        return 0
        ;;
      *)
        rm -rf -- "$STATE_DIR"
        ;;
    esac
  fi
}
trap cleanup_ephemeral_state EXIT

if [ ! -f "$CONFIG_FILE" ]; then
  STATE_DIR="$STATE_DIR" python3 - <<'PY'
import json
import os
from pathlib import Path
from datetime import datetime, timezone

state_dir = Path(os.environ["STATE_DIR"])
config = {
    "skillName": "make-ui-changes",
    "stateDir": str(state_dir),
    "auditDir": str(state_dir / "audits"),
    "patchDir": str(state_dir / "patches"),
    "updateDir": str(state_dir / "updates"),
    "tmpDir": str(state_dir / "tmp"),
    "logFile": str(state_dir / "runs.jsonl"),
    "branchPrefix": "ui-safe-change",
    "maxFilesChanged": 2,
    "maxTargetFilesForLowRisk": 2,
    "maxLinesChangedSoftLimit": 80,
    "retentionDays": 30,
    "autoUpdate": False,
    "autoUpdatePrompted": False,
    "updateSource": None,
    "autoCreatePr": False,
    "createdAt": datetime.now(timezone.utc).isoformat()
}
(state_dir / "config.json").write_text(json.dumps(config, indent=2) + "\n")
PY
fi

CONFIG_FILE="$CONFIG_FILE" python3 - <<'PY'
import json
import os
from pathlib import Path

cfg = json.loads(Path(os.environ["CONFIG_FILE"]).read_text())
for key in ["stateDir", "auditDir", "patchDir", "updateDir", "tmpDir"]:
    Path(cfg[key]).mkdir(parents=True, exist_ok=True)
Path(cfg["logFile"]).touch()
print("CONFIG_OK=true")
print(f"STATE_DIR={cfg['stateDir']}")
print(f"AUDIT_DIR={cfg['auditDir']}")
print(f"PATCH_DIR={cfg['patchDir']}")
print(f"UPDATE_DIR={cfg['updateDir']}")
print(f"LOG_FILE={cfg['logFile']}")
PY

CONFIG_FILE="$CONFIG_FILE" python3 - <<'PY' || true
import json
import os
from pathlib import Path
from datetime import datetime, timedelta, timezone

cfg = json.loads(Path(os.environ["CONFIG_FILE"]).read_text())
state_dir = Path(cfg["stateDir"]).resolve()
retention_days = int(cfg.get("retentionDays", 30))
cutoff = datetime.now(timezone.utc) - timedelta(days=retention_days)

for path in sorted(state_dir.rglob("*"), reverse=True):
    try:
        resolved = path.resolve()
        if resolved != state_dir and state_dir not in resolved.parents:
            continue
        if path.is_file():
            modified = datetime.fromtimestamp(path.stat().st_mtime, timezone.utc)
            if modified < cutoff:
                path.unlink()
        elif path.is_dir() and path != state_dir:
            try:
                path.rmdir()
            except OSError:
                pass
    except OSError:
        pass
PY

BRANCH="$(git branch --show-current 2>/dev/null || echo detached)"
HEAD_SHA="$(git rev-parse HEAD 2>/dev/null || echo unknown)"

RUN_ID="$RUN_ID" \
REPO_ROOT="$REPO_ROOT" \
BRANCH="$BRANCH" \
HEAD_SHA="$HEAD_SHA" \
STATE_MODE="$STATE_MODE" \
STATE_DIR="$STATE_DIR" \
LOG_FILE="$LOG_FILE" \
python3 - <<'PY'
import json
import os
from datetime import datetime, timezone
from pathlib import Path

event = {
    "ts": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
    "runId": os.environ["RUN_ID"],
    "event": "started",
    "repoRoot": os.environ["REPO_ROOT"],
    "branch": os.environ["BRANCH"],
    "head": os.environ["HEAD_SHA"],
    "stateMode": os.environ["STATE_MODE"],
    "stateDir": os.environ["STATE_DIR"],
}

with Path(os.environ["LOG_FILE"]).open("a", encoding="utf-8") as fh:
    fh.write(json.dumps(event, separators=(",", ":")) + "\n")
PY

echo "RUN_ID=$RUN_ID"
echo "REPO_ROOT=$REPO_ROOT"
echo "BRANCH=$BRANCH"
echo "HEAD=$HEAD_SHA"
echo "STATE_MODE=$STATE_MODE"
echo "STATE_DIR=$STATE_DIR"
echo "CONFIG_FILE=$CONFIG_FILE"
echo "AUDIT_DIR=$AUDIT_DIR"
echo "PATCH_DIR=$PATCH_DIR"
echo "UPDATE_DIR=$UPDATE_DIR"

```

If bootstrap prints BLOCKED, stop and report the blocker.

If STATE_MODE=ephemeral, do not create persistent audit or patch files.
Keep audit and patch data in memory where possible.
If temporary files are needed during execution, create them under a temporary run directory and delete that directory before exit.
Do not include Audit saved or Patch report paths in the final response.

And for cleanup:

``` bash
TMP_RUN_DIR="$(mktemp -d)"
trap 'rm -rf -- "$TMP_RUN_DIR"' EXIT

```
---

## 3. Resolve Active Skill Version

Version is read from the active skill file.

Use the active skill path if the harness provides it.
If no active skill path is available, search only likely local skill locations.

```bash
python3 - <<'PY'
from pathlib import Path
import re

candidates = []
for p in [
    Path("SKILL.md"),
    Path(".cursor/skill/make-ui-changes/SKILL.md"),
    Path(".cursor/skills/make-ui-changes/SKILL.md"),
]:
    if p.exists():
        candidates.append(p)

version = None
skill_file = None
for p in candidates:
    text = p.read_text(errors="replace")
    name_ok = re.search(r"(?m)^name:\s*make-ui-changes\s*$", text)
    m = re.search(r"(?m)^version:\s*([^\s]+)\s*$", text)
    if name_ok and m:
        version = m.group(1)
        skill_file = str(p)
        break

print(f"ACTIVE_SKILL_FILE={skill_file or ''}")
print(f"ACTIVE_SKILL_VERSION={version or ''}")
PY
```

If no active skill file can be found, continue without update checks and log:

```json
{"event":"update_check_skipped","reason":"active_skill_file_not_found"}
```

Do not guess the active skill file.
Guessing is how tools start editing their own shadow.

---

## 4. Update Preflight

The update flow has two separate decisions:

1. Is update configured?
2. Is access verified?

Do not check remote version until both are true.

### 4.1 Update Configuration Check

Read `config.json`.

If `updateSource` is null:

```text
Update check skipped: no update source configured.
```

Log and continue to the UI-change audit.

If `updateSource` exists but is invalid, log and continue without updating.
Do not partially repair it.
Do not prompt for credentials.
Do not ask the user to paste secrets.

Required fields for `type: github`:

```text
type, host, owner, repo, path, branch, auth
```

Required fields for `type: git`:

```text
type, repo, path, branch, auth
```

### 4.2 GitHub CLI Access Check

For private team repositories, prefer `type: github` with `auth: gh`.

Before any remote version check, run:

```bash
python3 - <<'PY'
import json
import os
from pathlib import Path
cfg = json.loads(Path(os.environ.get("CONFIG_FILE", "config.json")).read_text())
src = cfg.get("updateSource")
if not src:
    print("UPDATE_CONFIGURED=false")
else:
    print("UPDATE_CONFIGURED=true")
    print(f"UPDATE_TYPE={src.get('type','')}")
    print(f"UPDATE_AUTH={src.get('auth','')}")
    print(f"UPDATE_HOST={src.get('host','github.com')}")
PY

if command -v gh >/dev/null 2>&1; then
  GH_INSTALLED=true
else
  GH_INSTALLED=false
fi

echo "GH_INSTALLED=$GH_INSTALLED"

if [ "$GH_INSTALLED" = "true" ]; then
  if gh auth status --hostname github.com >/dev/null 2>&1; then
    echo "GH_AUTH_OK=true"
  else
    echo "GH_AUTH_OK=false"
  fi
else
  echo "GH_AUTH_OK=false"
fi
```

Rules:

```text
IF updateSource is missing:
  skip update check and continue.

IF updateSource.type == github AND updateSource.auth == gh:
  IF gh is not installed:
    skip update check, log reason gh_not_installed, continue.
  IF gh auth status fails:
    skip update check, log reason gh_not_authenticated, continue.
  ELSE:
    remote access preflight may continue.

IF updateSource.type == git AND updateSource.auth == ssh:
  run SSH preflight before remote access.

IF auth mode is unsupported:
  skip update check, log reason unsupported_update_auth, continue.
```

Update failure must not block a normal UI audit.
The skill is for UI changes. Updating itself is maintenance, not the mission.

### 4.3 GitHub Repository Access Check

When using `gh`, verify repository access before fetching the skill file.

```bash
gh repo view "OWNER/REPO" --json nameWithOwner,visibility >/dev/null 2>&1
```

If it fails:

```text
Update check skipped: GitHub CLI is authenticated, but this account cannot access OWNER/REPO.
```

Log and continue.

### 4.4 Fetch Remote Skill File With gh

Only after config and access checks pass, fetch the remote skill file from the configured `updateSource`.
Do not hardcode owner, repo, branch, or path in this section.
Read them from `config.json`.

```bash
python3 - <<'PY' > "$TMP_RUN_DIR/update-source.env"
import json
import shlex
from pathlib import Path
cfg = json.loads(Path(CONFIG_FILE).read_text())
src = cfg.get("updateSource") or {}
required = ["owner", "repo", "path", "branch"]
missing = [k for k in required if not src.get(k)]
if missing:
    print("UPDATE_SOURCE_VALID=false")
    print(f"UPDATE_SOURCE_ERROR={shlex.quote('missing_' + '_'.join(missing))}")
else:
    print("UPDATE_SOURCE_VALID=true")
    print(f"OWNER={shlex.quote(src['owner'])}")
    print(f"REPO={shlex.quote(src['repo'])}")
    print(f"PATH_IN_REPO={shlex.quote(src['path'])}")
    print(f"BRANCH={shlex.quote(src['branch'])}")
PY

. "$TMP_RUN_DIR/update-source.env"

if [ "${UPDATE_SOURCE_VALID:-false}" != "true" ]; then
  echo "Update check skipped: invalid updateSource config: ${UPDATE_SOURCE_ERROR:-unknown_error}"
else
  OUT_FILE="$UPDATE_DIR/SKILL.remote.$RUN_ID.md"
  ERR_FILE="$TMP_RUN_DIR/gh-api-fetch.err"
  if gh api \
    -H "Accept: application/vnd.github.raw" \
    "repos/$OWNER/$REPO/contents/$PATH_IN_REPO?ref=$BRANCH" \
    > "$OUT_FILE" 2> "$ERR_FILE"; then
    if [ ! -s "$OUT_FILE" ]; then
      rm -f "$OUT_FILE"
      echo "Update check skipped: remote skill file was empty."
    else
      echo "REMOTE_SKILL_FILE=$OUT_FILE"
    fi
  else
    STATUS=$?
    rm -f "$OUT_FILE"
    if grep -qi "Not Found" "$ERR_FILE"; then
      echo "Update check skipped: remote skill file was not found or the authenticated GitHub account cannot access it."
      echo "Checked: $OWNER/$REPO:$BRANCH:$PATH_IN_REPO"
    else
      echo "Update check skipped: failed to fetch remote skill file with gh api."
      echo "gh exit code: $STATUS"
    fi
  fi
fi
```

Then validate the downloaded file before using it.

Validation runs only when REMOTE_SKILL_FILE is set and the file exists.

Required markers:

```text
name: make-ui-changes
version:
allowed-tools:
# make-ui-changes
Audit first.
Apply only when risk is LOW_RISK and allowedToApply is true.
```

If the remote file is missing, inaccessible, empty, or invalid:

Update check skipped: remote skill file could not be trusted.

Log the reason and continue without updating.

Do not block the normal UI-change workflow because the remote skill file is missing.
Do not create the missing remote file.
Do not change update configuration automatically.


### 4.5 Compare Versions

Only compare versions after a valid local version and valid remote file exist.

Use semantic version comparison.
If parsing fails, skip update and log `version_parse_failed`.

### 4.6 Auto-Update Preference

If a newer trusted remote version exists and `autoUpdatePrompted=false`, ask one decision question:

```text
D1 — make-ui-changes update preference
ELI10: This skill can update its own instructions from the team repository before it edits UI code. That keeps the safety rules current.
Stakes: Auto-update can change local skill behavior. Manual update can leave old guardrails in place.
Recommendation: A, if the update repo is owned by the team and reviewed.
Completeness: A=8/10, B=7/10
Pros / cons:
A) Enable auto-update from the configured team repo
  ✅ Keeps guardrails current.
  ✅ Reduces drift across team members.
  ❌ A bad reviewed update can temporarily block UI edits.

B) Keep manual update
  ✅ No automatic local skill changes.
  ✅ Easier to inspect every update.
  ❌ People forget. Then old instructions keep shambling around.

Net: Enable auto-update only for a trusted team-owned source.
```

Save the answer in `config.json`.

If `autoUpdate=false`, log `update_available_manual_only` and continue.

### 4.7 Apply Auto-Update Safely

Only update when all are true:

```text
updateSource is valid
access preflight passed
remote skill downloaded
remote skill validated
remote version > local version
autoUpdate == true
active skill file path is known
active skill file has no uncommitted local changes
```

Procedure:

1. Copy active skill file to `updates/SKILL.local.$RUN_ID.bak`.
2. Copy remote file to active skill file path.
3. Re-read active skill file and validate markers.
4. If validation fails, restore backup.
5. Log success or failure.
6. Continue to UI audit.

If any condition fails, report the specific reason in the log and continue without updating.

Do not ask for secrets.
Do not run `gh auth login` automatically.
Do not mutate git credential config.
Do not overwrite an untracked or modified skill file.
Do not fetch arbitrary URLs.

---

## 5. UI Change Workflow

Follow this order exactly:

```text
Step 0: Bootstrap state and repo preflight
Step 1: Resolve active skill version
Step 2: Run update preflight if configured
Step 3: Understand user request
Step 4: Resolve target file
Step 5: Audit change
Step 6: Save audit JSON
Step 7: Stop unless audit allows apply
Step 8: Create safe branch
Step 9: Apply smallest UI-only edit
Step 10: Verify diff safety
Step 11: Run available lightweight checks
Step 12: Save patch report
Step 13: Report result
```

The update flow is not allowed to edit application code.
The audit flow is not allowed to edit files.
The apply flow is not allowed to exceed the audit scope.

---

## 6. Allowed LOW_RISK Changes

A change may be LOW_RISK only when all are true:

- One target UI file, or one target UI file plus one companion file when the change matches an allowed LOW_RISK exception.
- UI-only.
- No business logic.
- No state management.
- No data fetching.
- No validation behavior.
- No routing behavior.
- No auth or permission behavior.
- No dependency changes.
- No build/config changes.
- No generated files.
- No shared contract changes.
- No hidden refactor.
- Target confidence is high.


LOW_RISK may touch 2 files only when one of these narrow cases applies:
  1. LOW_RISK_I18N_ADD:
    - one file is the target UI template/component markup
    - the second file is the default translation file, usually `en.json`
    - the translation file only receives new keys
    - no existing translation keys are changed, deleted, renamed, or moved
    - no migration is needed

  2. LOW_RISK_ANGULAR_COMPANION_TS:
    - one file is the Angular template, usually *.component.html
    - the second file is the matching Angular component class, usually *.component.ts
    - the TypeScript change is purely presentational support for the template
    - allowed TypeScript changes are limited to adding or adjusting static readonly display values, static option labels, simple view constants, or imports already required for template-only presentation
    - no handlers, signals, observables, services, inputs, outputs, forms, routing, permissions, API calls, computed business logic, validation, or state transitions are changed
    - no constructor or dependency injection changes are made

- template uses the new key through the existing translation pattern
- no TypeScript logic changes
- no migration is needed
- placeholder text
- static aria text
- Tailwind classes
- minor spacing/layout polish inside one file
- visible copy only when the project does not use i18n for that area
- static aria text through translation keys when the surrounding file already uses i18n

### Ownership Boundary Rule

The skill must prefer fixing the issue inside the domain, stream, feature, or page area where the reported UI belongs.

A request from any PM or team should be resolved in the local owning area of the reported page when possible.

Shared UI/design-library files, such as files under `/frontend/dashboard/projects/prewave-ui/**`, are treated as separately owned code.

A shared UI/design-library file may be changed automatically only when all are true:
- the audit shows the issue belongs to the shared component itself,
- exactly one shared UI/design-library file is changed,
- no domain, stream, page, or feature file is changed in the same patch,
- the change is UI-only,
- the change does not alter component API, behavior, state, styling contract, design tokens, or public usage expectations,
- the final response says the PR should be reviewed by the shared UI/design-library owners.

If the fix requires both a domain/page file and a shared UI/design-library file, classify as MEDIUM_RISK and do not apply automatically.

If the issue can be fixed either locally or in the shared UI/design-library, prefer the local domain/page fix unless the shared component is clearly defective.

### i18n rule:
If the project uses translation files for the target UI, do not inline new user-facing text in the template.

Use the existing translation mechanism.
Allowed:
- add new key to en.json in the corresponding area
- use key in template with existing translate pipe/pattern

Blocked:
- modify existing translation values
- delete or rename translation keys
- edit non-default locale files unless configured
- create or modify migration files
- change translation loading/configuration

---

## 7. Refusal Categories

### 7.1 MEDIUM_RISK

Use when the request might be frontend-only but needs engineering judgment.

Examples:

- Multiple likely files.
- Shared component likely affected.
- Props contract may change.
- Conditional rendering may change.
- Responsive redesign across a page.
- Large visual redesign.
- Diff likely exceeds configured soft limit.

Result:

```text
Result:

Applied: no.
Reason: MEDIUM_RISK.

Create a Jira-style engineering ticket from the audit result and return it.

Ticket format:

Title:
[Infer a short, specific title from the requested change and affected UI area]

Problem:
[Explain the problem in one or two paragraphs only]

Solution:
[Describe the recommended solution in one or two paragraphs, keep it at a high level, implementation-aware but not overly prescriptive. The solution must be based on the actual audit findings.]

Acceptance criteria:
[Write specific, testable criteria for this request, Each criterion in a bullet point must describe an observable outcome or verification point.]
Do not use generic acceptance criteria that could apply to every ticket.
```

### 7.2 HIGH_RISK

Use when product behavior, data, security, architecture, or backend contracts are involved.

Examples:

- permissions
- authentication
- role-based visibility
- API calls
- form submission
- validation
- state changes
- routing
- backend changes
- database/schema changes
- dependency changes

Result:

```text
Applied: no.
Reason: HIGH_RISK.
Next safe action: engineering review required, create a ticket with format mentioned in [7.1 MEDIUM_RISK](#71-medium_risk).
```

### 7.3 BLOCKED

Use when the skill cannot safely decide.

Examples:

- Not a git repository.
- Target file unclear.
- Multiple plausible targets and no evidence-based winner.
- Target file has existing user changes.
- Required source folder missing.
- Active skill file cannot be safely updated.

Result:

```text
Applied: no.
Reason: BLOCKED.
Next safe action: provide the missing route, file path, visible text, or resolve the local repo state.
```

No category should produce application code changes except LOW_RISK.

---

## 8. Understand the Request

Extract:

- requested visual change
- exact copy/text if any
- target route/page/component/file/folder
- nearby visible text
- requested action: audit only or audit and apply

If the target is unclear, search first.
If search does not identify one target with high confidence, ask one focused question.

Good question:

```text
Which page or component should this change apply to? Give the route, file path, or visible text near the area.
```

Weak question:

```text
Can you provide more details?
```

That question is too vague to act on. Use something specific

---

## 9. Resolve Target File

Use `Glob`, `Grep`, `Read`, and `Jetbrains` when available for code navigation, symbol lookup, or project inspection.

Search order:

1. Exact visible text.
2. Route path.
3. Component name.
4. Page folder.
5. Nearby labels/headings.

Ignore:

```text
node_modules/
dist/
build/
coverage/
out/
.git/
generated/
```

Useful command:

```bash
find . -type f \( \
  -name '*.ts' -o \
  -name '*.tsx' -o \
  -name '*.html' -o \
  -name '*.css' \
  -name '*.scss' \
\) \
-not -path './node_modules/*' \
-not -path './dist/*' \
-not -path './build/*' \
-not -path './.next/*' \
-not -path './coverage/*' \
| head -200
```

If one file clearly owns the requested UI, continue.
If not, stop as BLOCKED and ask one focused question.

---

## 10. Audit JSON

Before editing, write audit JSON to:

```text
<auditDir>/<RUN_ID>.audit.json
```

Shape:

```json
{
  "runId": "string",
  "timestamp": "ISO-8601",
  "request": {
    "raw": "string",
    "summary": "string",
    "requestedAction": "audit_only | audit_and_apply"
  },
  "repository": {
    "root": "string",
    "branch": "string",
    "head": "string",
    "dirty": false
  },
  "skill": {
    "name": "make-ui-changes",
    "version": "read-from-active-skill-file"
  },
  "target": {
    "confidence": "high | medium | low",
    "files": ["string"],
    "primaryFile": "string | null",
    "evidence": ["string"]
  },
  "classification": {
    "risk": "LOW_RISK | MEDIUM_RISK | HIGH_RISK | BLOCKED",
    "allowedToApply": false,
    "changeType": "copy | style | markup | layout | accessibility-copy | logic | unknown",
    "estimatedFilesChanged": 0,
    "estimatedLinesChanged": 0,
    "requiresLogicChange": false,
    "requiresStateChange": false,
    "requiresDataFlowChange": false,
    "requiresMultipleFiles": false,
    "requiresSharedComponentChange": false,
    "requiresDependencyChange": false,
    "requiresBackendChange": false,
    "requiresTestRewrite": false
  },
  "decision": {
    "summary": "string",
    "reasons": ["string"],
    "stopReason": "string | null",
    "nextSafeAction": "string"
  },
  "ownership": {
    "reportedArea": "string",
    "preferredChangeArea": "domain_or_page | shared_ui_library | unknown",
    "touchesDomainOrPageArea": false,
    "touchesSharedUiLibrary": false,
    "ownershipBoundary": "local_domain_only | single_shared_ui_library_file | mixed_ownership | unknown",
    "requiredReviewOwner": "owning domain team | shared UI/design-library owners | unknown"
  }
}
```

### 10.1 allowedToApply Rule

Set `allowedToApply=true` only when:

```text
risk == LOW_RISK
AND target.confidence == high
AND target.files.length <= 2
AND estimatedFilesChanged <= maxFilesChanged
AND requiresLogicChange == false
AND requiresStateChange == false
AND requiresDataFlowChange == false
AND requiresMultipleFiles == false
AND requiresDependencyChange == false
AND requiresBackendChange == false
AND requiresTestRewrite == false
AND ownership.ownershipBoundary != mixed_ownership
AND (
  ownership.ownershipBoundary == local_domain_only
  OR ownership.ownershipBoundary == single_shared_ui_library_file
)
```

Every other case is `allowedToApply=false`.

---

## 11. Stop Gate

After saving audit JSON:

```text
IF allowedToApply != true:
  report audit result
  do not edit application files
  do not create a change branch
  stop
```

The stop gate is not advisory.
It is the guardrail.

---

## 12. Create Safe Branch

Only after LOW_RISK audit approval.

Check current state:

```bash
git status --short
```

If the target file has existing local changes, stop as BLOCKED.
Do not overwrite user work.

Create a branch:

```bash
python3 - <<'PY'
import json
import os
import re
from pathlib import Path
cfg = json.loads(Path(os.environ.get("CONFIG_FILE", "config.json")).read_text())
summary = "ui-change"
slug = re.sub(r"[^a-z0-9]+", "-", summary.lower()).strip("-")[:40] or "ui-change"
print(f"{cfg.get('branchPrefix', 'ui-safe-change')}/{slug}")
PY
```

Then:

```bash
git checkout -b "<branch-from-command>"
```

If branch creation fails, stop and report the git error.

Do not commit directly to the current branch.

---

## 13. Apply the Change

Use the smallest possible edit.

Preferred order:

1. exact copy replacement
2. Tailwind class edit
3. static markup edit
4. static accessibility copy edit

Do not rewrite the component.
Do not reformat the file.
Do not rename props.
Do not move files.
Do not change imports for convenience.
Do not touch tests as part of this skill.
Do not touch package files.
Do not touch lock files.

If the implementation requires more than the audit allowed, revert partial changes and reclassify as MEDIUM_RISK or HIGH_RISK.

---

## 14. Verify Diff Safety

After editing:

```bash
git diff --stat
git diff --check
git diff -- "<primary-file>"
```

Validate:

- <=2 files changed
- target files only
- UI-only diff
- no behavior changes
- no unrelated formatting churn
- no package or lock files
- no generated files

Ownership diff validation:
- If changed files include both domain/page files and shared UI/design-library files, reclassify as MEDIUM_RISK and revert.
- If only one shared UI/design-library file changed, verify the audit classified it as `single_shared_ui_library_file`.
- If a shared UI/design-library file changed, final response must say the changes needs review from the shared UI/design-library owners.

Suspicious diff terms:

```text
signal
ngrx
useState
useEffect
useMemo
useCallback
fetch
axios
query
mutation
router
navigate
redirect
auth
permission
role
schema
validation
zod
yup
submit
onSubmit
package.json
lock
```

If unsafe diff is found:

```bash
git checkout -- "<primary-file>"
```

Then report the reclassified risk.

---

## 15. Run Lightweight Checks

Detect package manager and scripts from existing files only.

```bash
if [ -f package.json ]; then
  node -e "const p=require('./package.json'); console.log(JSON.stringify(p.scripts || {}, null, 2))"
fi
```

Allowed checks:

```text
lint
typecheck
test
```

Run only scripts that already exist.
Do not install dependencies.
Do not add scripts.
Do not modify configuration.

If checks are unavailable, mark checks as skipped with reason.
If checks fail because of this change, revert and report.
If checks fail for existing unrelated reasons, report evidence without hiding it.

---

## 16. Patch Report

Write patch report to:

```text
<patchDir>/<RUN_ID>.patch.json
```

Shape:

```json
{
  "runId": "string",
  "timestamp": "ISO-8601",
  "auditPath": "string",
  "patch": {
    "branch": "string",
    "filesChanged": ["string"],
    "summary": "string",
    "diffStat": "string"
  },
  "checks": {
    "commandsRun": ["string"],
    "passed": true,
    "skipped": false,
    "notes": ["string"]
  },
  "pullRequest": {
    "created": false,
    "url": null,
    "notes": "PR creation is outside this skill unless autoCreatePr is enabled and gh preflight passes."
  }
}
```

Append completion event to `runs.jsonl`.

---

## 17. PR Handling

This skill does not create a PR by default.

If `autoCreatePr=true`, PR creation must pass preflight first:

```bash
command -v gh >/dev/null 2>&1
gh auth status --hostname github.com >/dev/null 2>&1
git remote get-url origin >/dev/null 2>&1
```

If any preflight fails, report:

```text
PR not created: <specific preflight failure>.
```

If all pass, use:

```bash
git push -u origin "<branch>"
gh pr create --fill
```

Do not invent PR URLs.
Do not continue PR creation after push failure.

---

## 18. Final Response Format

### 18.1 Not Applied

```text
Audit result: <MEDIUM_RISK | HIGH_RISK | BLOCKED>
Applied: no
Reason: <specific reason>
Audit saved: <path>
Next safe action: <specific next step>
```

### 18.2 Applied

```text
Audit result: LOW_RISK
Applied: yes
Branch: <branch>
Changed file: <file>
Summary: <what changed>
Checks: <passed/skipped/failed with command names>
Audit saved: <path>
Patch report: <path>
```

Keep it short.
Do not dump JSON unless requested.

---

## 19. Operating Principle

Small UI changes should leave a clean trail:

- what was asked,
- what was inspected,
- why it was safe or unsafe,
- what changed,
- what checks ran,
- where the audit and patch report live.

When access is missing, report missing access.
When the target is unclear, ask one precise question.
When the request is too broad, refuse the application and explain the safe next step.
When it is truly LOW_RISK, make the smallest correct change.

## 20. Maintenance Note

This skill is intentionally kept as one file for now so the team can test the full workflow without chasing logic across hooks, helper scripts, or reference files.

The skill may create local state files for audit reports, patch reports, update checks, temporary files, and run logs. These files are kept locally, under an ignored workspace folder when available, and are not meant to be committed. They do not intentionally identify the user or collect personal details.

State files exist so users and maintainers may debug what happened during a run:
- what was requested,
- what was audited,
- why the skill applied or refused a change,
- what files changed,
- what checks ran.

Old local state files are deleted automatically based on the configured retention period. After the workflow is proven, large sections may be moved into separate files or scripts.

Team: Super Compliance (SC)
Maintainer: James

For questions:
- Ask the maintainer for workflow or implementation questions.
- Ask the team for adoption, review, or long-term changes.
