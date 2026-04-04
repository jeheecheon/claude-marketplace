Create a GitHub pull request for the current branch.

# Context

- Base branch: **develop**

# Decision Tree

> **Rule: Follow every step of the decision tree in order. Do not skip or reorder steps unless the user explicitly instructs otherwise.**

```
START
│
├─ [A0] Check current branch
│  │  Run: git branch --show-current
│  ├─ On a non-default branch (not develop, not main) → [A3]
│  └─ On develop or main → need to create branch → [A1]
│
├─ [A1] Fetch remote branches
│  │  Run: git branch -r
│  └─ Save output → [A2]
│
├─ [A2] Determine branch naming convention & create branch
│  │  Analyze branch names from A1
│  ├─ ≥ 5 remote branches → check for consistent pattern
│  │  ├─ Consistent convention detected → create branch following detected convention → [A3]
│  │  └─ No consistent convention → create branch with default naming → [A3]
│  └─ < 5 remote branches → create branch with default naming → [A3]
│
├─ [A3] Fetch commit history since develop
│  │  Run: git log develop..HEAD --oneline
│  ├─ Commits exist → [A4]
│  └─ No commits vs develop → inform user, nothing to PR → STOP
│
├─ [A4] Fetch change stats since develop
│  │  Run: git diff develop...HEAD --stat
│  └─ Save output → [A5]
│
├─ [A5] Check remote tracking status
│  │  Run: git status -sb
│  ├─ Branch pushed & up-to-date → [A7]
│  └─ Not pushed or has unpushed commits → [A6]
│
├─ [A6] Push branch to remote
│  │  Run: git push -u origin <branch>
│  ├─ OK → [A7]
│  └─ FAIL → show error → STOP
│
├─ [A7] Check for PR template
│  │  Check: .github/pull_request_template.md exists?
│  ├─ Exists → read template, save structure → [A10]
│  └─ Does not exist → [A8]
│
├─ [A8] Fetch merged PRs
│  │  Run: gh pr list --state merged --limit 5 --json title,body
│  └─ Save output → [A9]
│
├─ [A9] Determine PR convention
│  │  Analyze merged PRs from A8
│  ├─ ≥ 3 merged PRs → check for consistent pattern
│  │  ├─ Consistent convention detected → save detected convention → [A10]
│  │  └─ No consistent convention → save default convention → [A10]
│  └─ < 3 merged PRs → save default convention → [A10]
│
├─ [A10] Draft PR title & body
│  └─ Show drafted title and body to user → [A11]
│
├─ [A11] Confirm with user
│  ├─ User approves → [A12]
│  └─ User requests changes → revise draft → [A11]
│
├─ [A12] Create PR
│  │  Run: gh pr create --title "<title>" --body "<body>" --base develop
│  ├─ OK → [A13]
│  └─ FAIL → show error → STOP
│
└─ [A13] Report
     Show PR URL returned by gh pr create
     Confirm PR created successfully → END
```

# Actions

## A0. Check Current Branch

```bash
git branch --show-current
```

Read the output. Determine the current branch name.

- If on a **non-default branch** (not develop, not main) → continue to A3.
- If on **develop** or **main** → need to create a branch. Continue to A1.

## A1. Fetch Remote Branches

```bash
git branch -r
```

Read and save the output. Count the number of remote branches. This is used in A2 to detect naming convention.

## A2. Determine Branch Naming Convention & Create Branch

Analyze the branch names from A1. Check the following aspects for a consistent pattern:

- **Prefix style**: what prefixes are used (e.g. `feature/`, `feat/`, `fix/`, `bugfix/`)
- **Separator**: what separates prefix from description (`/`, `-`, `_`)
- **Description casing**: kebab-case, snake_case, camelCase, or other

Decision:

- **≥ 5 remote branches, consistent convention detected** → extract the full pattern. Create a branch following the detected convention.
- **≥ 5 remote branches, no consistent convention** OR **< 5 remote branches** → use the default naming:
  - Format: `[type]/<short-kebab-description>`
  - Types: `feature`, `fix`, `refactor`, `chore`

Create the branch:

```bash
git checkout -b <branch-name>
```

Choose a name based on the most recent commit messages. Continue to A3.

## A3. Fetch Commit History Since Develop

```bash
git log develop..HEAD --oneline
```

Read and save the output. This shows all commits that will be included in the PR.

- If **commits exist** → continue to A4.
- If **no commits vs develop** → inform the user there is nothing to create a PR for. **STOP.**

## A4. Fetch Change Stats Since Develop

```bash
git diff develop...HEAD --stat
```

Read and save the output. This shows the full scope of file changes. Together with A3, this is the foundation for the PR title and body in A10.

## A5. Check Remote Tracking Status

```bash
git status -sb
```

Read the output. Determine whether the branch is pushed and up-to-date with the remote.

- If branch **tracks a remote and is up-to-date** → continue to A7.
- If branch is **not pushed or has unpushed commits** → continue to A6.

## A6. Push Branch to Remote

```bash
git push -u origin <branch>
```

- If push **succeeds** → continue to A7.
- If push **fails** → show the error output. **STOP.**

## A7. Check for PR Template

Check whether a PR template exists:

```bash
cat .github/pull_request_template.md
```

- If template **exists** → read its content. Save the structure for use in A10. Skip A8 and A9.
- If template **does not exist** → continue to A8.

## A8. Fetch Merged PRs

```bash
gh pr list --state merged --limit 5 --json title,body
```

Read and save the output. Count the number of merged PRs returned. This is used in A9 to detect convention.

## A9. Determine PR Convention

Analyze the merged PRs from A8. Check the following aspects for a consistent pattern:

**Title:**
- **Prefix type**: whether titles use a type prefix (e.g. `feat:`, `fix:`)
- **Scope**: whether titles include a scope (e.g. `feat(auth):` vs `feat:`)
- **Separator**: what follows the type/scope (e.g. `:`, `-`, space)
- **Casing**: uppercase or lowercase first letter of summary

**Body:**
- **Language**: Korean, English, or mixed
- **Section structure**: what sections exist (e.g. WHY/WHAT, Summary/Changes, Description/Checklist)
- **Section headers**: format of headers (e.g. `## Summary`, `### Changes`, `**WHY**`)
- **Issue linking**: whether and how issues are linked (e.g. `Closes #N`, `Fixes #N`, `Refs #N`, or none)

Decision:

- **≥ 3 merged PRs, consistent convention detected** → extract the full pattern (title format + body structure + language + issue linking). Save it for A10.
- **≥ 3 merged PRs, no consistent convention** OR **< 3 merged PRs** → save the default convention for A10:
  - Title format: `<type>: <summary>` (feat, fix, refactor, chore)
  - Type inference: branch prefix → type (`feature/` → `feat`, `fix/` → `fix`, `refactor/` → `refactor`, other → `chore`)
  - Body language: Korean
  - Body structure: WHY (problem/motivation) + WHAT (implementation decisions)
  - Issue linking: `Closes #N` or `Refs #N` when applicable

## A10. Draft PR Title & Body

Apply the convention determined in A7 or A9. Base the content on the full change analysis from A3 + A4.

**Title:**
- Draft a title following the detected or default convention.
- Follow all aspects detected in A9 (prefix type, scope, separator, casing).

**Body:**
- Follow the structure and language from the convention (template, detected pattern, or default).
- Follow the issue linking format from the convention.

Show the drafted title and body to the user.

## A11. Confirm with User

Wait for the user's response.

- If **user approves** → continue to A12.
- If **user requests changes** → revise the draft and show again. Repeat until approved.

## A12. Create PR

```bash
gh pr create --title "<title>" --body "<body>" --base develop
```

- If PR creation **succeeds** → continue to A13.
- If PR creation **fails** → show error output and **STOP.**

## A13. Report

Show the PR URL returned by `gh pr create`. Confirm the PR was created successfully.

# Input

No arguments. This command always follows the full decision tree.
