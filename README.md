# make-ui-changes

A guarded agent skill for making small, safe UI changes without turning a simple visual tweak into an accidental refactor.

This repository contains the canonical `SKILL.md` used by the team for testing and update checks.

## What this skill is for

Use this skill when the requested change is small, presentational, and local.

Good examples:

- update button text
- adjust helper text
- fix spacing
- tweak layout polish
- update Tailwind classes
- add a new translation key and use it in a template
- make a minor Angular template change

Bad examples:

- change permissions
- change routing
- change API calls
- change form behavior
- change validation
- update shared UI library and domain code in the same patch
- refactor a component
- redesign a full page

The skill audits first. It only applies the change when the result is `LOW_RISK`.

## Repository layout

Recommended layout for this test repo:

```text
make-ui-changes/
  README.md
  SKILL.md
```

If this skill is later moved into the company dashboard repo, the intended path is:

```text
ai/SC/make-ui-changes/SKILL.md
```

## Update source config

For this test repository, use:

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

For the company dashboard repository, use:

```json
{
  "type": "github",
  "host": "github.com",
  "owner": "prewave",
  "repo": "dashboard",
  "path": "ai/SC/make-ui-changes/SKILL.md",
  "branch": "main",
  "auth": "gh"
}
```

Prefer the `github` + `gh` option for private repositories. It avoids cloning the whole repository just to fetch one file, because apparently even one Markdown file can summon ceremony if left unsupervised.

## Required local tools

For update checks from private GitHub repositories, the user needs:

- Git
- GitHub CLI (`gh`)
- Cursor
- access to the repository that contains the skill

Authenticate GitHub CLI:

```bash
gh auth login
```

Verify authentication:

```bash
gh auth status --hostname github.com
```

GitHub CLI supports browser-based login and stores credentials for later CLI use. The skill checks `gh auth status` before trying to fetch private repository content.

## Test the remote fetch manually

For this test repo:

```bash
gh api \
  -H "Accept: application/vnd.github.raw" \
  "repos/james-aworo/make-ui-changes/contents/SKILL.md?ref=main" \
  | head
```

Expected result: the command prints the beginning of `SKILL.md`, including the frontmatter.

If it fails:

- check that the repo exists
- check that the file is actually pushed
- check that the branch is correct
- check `gh auth status --hostname github.com`
- check that the authenticated account has access

## Runtime state

The skill writes runtime state only when it finds a safe ignored workspace folder.

Preferred state path in the target project:

```text
.cursor/plans/make-ui-changes/state
```

State is local only and is not meant to be committed.

It may contain:

- audit reports
- patch reports
- update check files
- temporary files
- run logs

The state is kept so users and maintainers can debug what happened during a run. Old state is deleted based on the configured retention period.

If no safe ignored state folder exists, the skill runs in ephemeral mode and cleans up temporary files before exit.

## Git ignore requirement

In the target project, make sure runtime state is ignored:

```gitignore
.cursor/plans/
.cursor/tmp/
.tmp/
tmp/
```

At minimum, this must be ignored:

```gitignore
.cursor/plans/make-ui-changes/state/
```

Do not put runtime state under the skill folder itself. The skill file is shared project knowledge. Runtime files are local execution noise. Mixing them is how a safe UI tweak turns into a Git diff swamp.

## Ownership boundary

The skill should prefer fixing the issue inside the domain, stream, feature, or page area where the reported UI belongs.

Shared UI/design-library files are treated as separately owned code.

For the dashboard project, shared UI/design-library files include:

```text
/frontend/dashboard/projects/prewave-ui/**
```

A shared UI/design-library file may be changed automatically only when:

- the audit shows the issue belongs to the shared component itself
- exactly one shared UI/design-library file is changed
- no domain, stream, page, or feature file is changed in the same patch
- the change is UI-only
- the change does not alter component API, inputs, outputs, behavior, state, styling contract, design tokens, or public usage expectations

If the fix requires both a domain/page file and a shared UI/design-library file, the skill should classify it as `MEDIUM_RISK` and stop.

## Risk model

The skill uses these outcomes:

```text
LOW_RISK
MEDIUM_RISK
HIGH_RISK
BLOCKED
```

Only `LOW_RISK` can be applied automatically.

`MEDIUM_RISK` should return a short Jira-style ticket:

```text
Title
Problem
Solution
Acceptance criteria
```

The ticket content must be inferred from the audit result, not copied from a generic template.

## PM testing checklist

Before testing:

- [ ] Open the target project in Cursor.
- [ ] Confirm the project has the latest `SKILL.md`.
- [ ] Confirm GitHub CLI is authenticated with `gh auth status --hostname github.com`.
- [ ] Confirm the working tree is clean with `git status`.
- [ ] Start with a small UI-only request.

Good first test request:

```text
Use make-ui-changes to update the helper text on this page from "X" to "Y".
```

Expected behavior:

- the skill audits first
- the skill applies only if the result is `LOW_RISK`
- the skill refuses risky or unclear requests
- the skill keeps changes local to the correct ownership boundary
- the skill reports clearly what happened

## Notes for maintainers

This skill is intentionally kept as one file for now so the team can test the full workflow without chasing behavior across hooks, helper scripts, or reference files.

After the workflow is proven, these sections may be split later:

- update handling
- i18n rules
- audit schema
- bash helpers
- hooks 
- ownership boundary rules

