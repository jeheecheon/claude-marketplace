Create a GitHub issue based on user input or current branch changes.

# Context

- Rule: **1 issue = 1 problem/feature. No exceptions.**

# Decision Tree

```
START
│
├─ [A0] Check input
│  ├─ User provided description → save description → [A3]
│  └─ No input → need to infer from changes → [A1]
│
├─ [A1] Detect base branch
│  │  Run: git branch -r
│  ├─ develop branch exists → use develop as base → [A2]
│  └─ No develop branch → use main as base → [A2]
│
├─ [A2] Gather all changes
│  │  Run: git diff <base>...HEAD
│  │  Run: git diff --staged
│  │  Run: git diff
│  │  Run: git status
│  └─ Save all outputs → [A3]
│
├─ [A3] Fetch existing issues
│  │  Run: gh issue list --state all --limit 10 --json title,body,labels
│  └─ Save output → [A4]
│
├─ [A4] Determine issue convention
│  │  Analyze issues from A3
│  ├─ ≥ 5 issues → check for consistent pattern
│  │  ├─ Consistent convention detected → save detected convention → [A5]
│  │  └─ No consistent convention → save default convention → [A5]
│  └─ < 5 issues → save default convention → [A5]
│
├─ [A5] Fetch available labels
│  │  Run: gh label list --json name,description
│  └─ Save output → [A6]
│
├─ [A6] Draft issue title, body & labels
│  └─ Show draft to user → [A7]
│
├─ [A7] Confirm with user
│  ├─ User approves → [A8]
│  └─ User requests changes → revise draft → [A7]
│
├─ [A8] Create issue
│  │  Run: gh issue create --title "<title>" --body "<body>" --label "<labels>"
│  ├─ OK → [A9]
│  └─ FAIL → show error → STOP
│
└─ [A9] Report
     Show issue URL returned by gh issue create
     Confirm issue created successfully → END
```

# Actions

## A0. Check Input

Determine whether the user provided a description of the issue.

- If **user provided description** (natural language, bug report, feature request, etc.) → save it as the basis for drafting. Skip to A3.
- If **no input** → need to infer the issue from current code changes. Continue to A1.

## A1. Detect Base Branch

```bash
git branch -r
```

Read the output. Determine which base branch to compare against.

- If **develop** branch exists on remote → use `develop` as base. Continue to A2.
- If **no develop** branch → use `main` as base. Continue to A2.

## A2. Gather All Changes

Run all four commands and read their output:

```bash
git diff <base>...HEAD
git diff --staged
git diff
git status
```

This covers:
- **Committed changes** since base branch (`git diff <base>...HEAD`)
- **Staged changes** not yet committed (`git diff --staged`)
- **Unstaged changes** in working tree (`git diff`)
- **Untracked files** and overall status (`git status`)

Save all outputs. These are used in A6 to understand the problem context and draft the issue.

## A3. Fetch Existing Issues

```bash
gh issue list --state all --limit 10 --json title,body,labels
```

Read and save the output. Count the number of issues returned. This is used in A4 to detect convention.

## A4. Determine Issue Convention

Analyze the issues from A3. Check the following aspects for a consistent pattern:

**Title:**
- **Prefix**: whether titles use a prefix or tag (e.g. `[Bug]`, `feat:`, `🐛`)
- **Casing**: uppercase or lowercase first letter
- **Style**: imperative (`Fix login`) vs descriptive (`Login is broken`) vs noun phrase (`Login bug`)

**Body:**
- **Language**: Korean, English, or mixed
- **Section structure**: what sections exist (e.g. Description/Steps to Reproduce, Summary/Details, Problem/Solution)
- **Section headers**: format of headers (e.g. `## Description`, `### Steps`, `**Problem**`)
- **List style**: bullet points (`-`) vs numbered (`1.`) vs prose

**Labels:**
- **Label usage**: whether issues consistently use labels
- **Label patterns**: common label categories (e.g. `bug`, `feature`, `enhancement`, `priority/*`)

Decision:

- **≥ 5 issues, consistent convention detected** → extract the full pattern (title format + body structure + language + label usage). Save it for A6.
- **≥ 5 issues, no consistent convention** OR **< 5 issues** → save the default convention for A6:
  - Title: plain descriptive summary, no prefix
  - Body language: Korean
  - Body structure:
    ```
    # Description
    - summary point 1
    - summary point 2

    # Steps to Reproduce
    1. step 1
    2. step 2
    ```
  - Labels: select the most appropriate from available labels in A5

## A5. Fetch Available Labels

```bash
gh label list --json name,description
```

Read and save the output. This is used in A6 to select appropriate labels.

- If **no labels exist** in the repository → skip label assignment in A6.
- If **labels exist** → continue to A6.

## A6. Draft Issue Title, Body & Labels

Apply the convention determined in A4. If no user input was provided, base the content on the change analysis from A2.

**Title:**
- Follow the detected or default convention.
- Must clearly describe the problem or feature in one line.

**Body:**
- Follow the structure and language from the convention.
- If based on code changes (A2): summarize what the changes reveal about the problem or feature.
- If based on user input (A0): structure the user's description into the convention's format.

**Labels:**
- Select the most appropriate labels from the list in A5.
- Match labels to the nature of the issue (bug, feature, enhancement, etc.).
- If no labels exist in the repo, omit the `--label` flag.

Show the drafted title, body, and labels to the user.

## A7. Confirm with User

Wait for the user's response.

- If **user approves** → continue to A8.
- If **user requests changes** → revise the draft and show again. Repeat until approved.

## A8. Create Issue

```bash
gh issue create --title "<title>" --body "<body>" --label "<label1>,<label2>"
```

Omit `--label` if no labels were selected.

- If issue creation **succeeds** → continue to A9.
- If issue creation **fails** → show error output and **STOP.**

## A9. Report

Show the issue URL returned by `gh issue create`. Confirm the issue was created successfully.

# Input

`$ARGUMENTS` — optional natural language description of the issue. If provided, use it as the basis for drafting (skip A1 and A2). If not provided, infer from current branch changes.
