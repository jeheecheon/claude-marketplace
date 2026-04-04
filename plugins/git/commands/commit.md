Create a git commit for staged changes, split by feature unit (one feature/task per commit).

# Context

- Rule: **1 feature/task = 1 commit. No exceptions.**
- Rule: **NEVER include `Co-Authored-By` or any AI attribution. Leave no trace of AI assistance in commits — no AI-related tags, footers, or metadata.**

# Decision Tree

> **Rule: Follow every step of the decision tree in order. Do not skip or reorder steps unless the user explicitly instructs otherwise.**

```
START
│
├─ [A0] Check working tree status
│  │  Run: git status
│  ├─ Staged changes exist → [A1]
│  └─ No staged changes → show output, suggest files to stage → STOP
│
├─ [A1] Read staged diff
│  │  Run: git diff --staged
│  └─ Save diff output → [A2]
│
├─ [A2] Fetch commit history
│  │  Run: git log --oneline -10
│  └─ Save output → [A3]
│
├─ [A3] Determine commit convention
│  │  Analyze commit messages from A2
│  ├─ ≥ 5 commits → check for consistent pattern
│  │  ├─ Consistent convention detected → save detected convention → [A4]
│  │  └─ No consistent convention → save default convention → [A4]
│  └─ < 5 commits → save default convention → [A4]
│
├─ [A4] Analyze staged changes
│  │  Review diff from A1
│  ├─ All changes belong to one feature/task → [A5]
│  └─ Changes span multiple features/tasks → STOP
│
├─ [A5] Draft commit message → [A6]
│
├─ [A6] Create commit
│  │  Run: git commit -m "<message>"
│  ├─ OK → [A7]
│  └─ FAIL → show error, diagnose cause → STOP
│
└─ [A7] Report
     Run: git log --oneline -1
     Show output, confirm commit created successfully → END
```

# Actions

## A0. Check Working Tree Status

```bash
git status
```

Read the output. Check whether staged changes exist.

- If **no staged changes**: show the output, suggest which files to stage. **STOP — do not proceed without staged changes.**
- If **staged changes exist** → continue to A1.

## A1. Read Staged Diff

```bash
git diff --staged
```

Read and save the full diff output. This is used in A4 to analyze changes and in A5 to draft an accurate message.

## A2. Fetch Commit History

```bash
git log --oneline -10
```

Read and save the output. Count the number of commits returned. This is used in A3 to detect convention.

## A3. Determine Commit Convention

Analyze the commit messages from A2. Check the following aspects for a consistent pattern:

- **Prefix type**: whether messages use a type prefix (e.g. `feat`, `fix`, `refactor`)
- **Scope**: whether messages include a scope (e.g. `feat(auth):` vs `feat:`)
- **Separator**: what follows the type/scope (e.g. `:`, `-`, space)
- **Casing**: uppercase or lowercase first letter of summary
- **Tense**: imperative (`add`) vs past tense (`added`) vs present (`adds`)
- **Summary length**: typical character count

Decision:

- **≥ 5 commits, consistent convention detected** → extract the full pattern. Save it for A5.
- **≥ 5 commits, no consistent convention** OR **< 5 commits** → save the default convention for A5:
  - Format: `[type]: [short summary]`
  - Types: `feat`, `fix`, `refactor`, `chore`
  - No scope
  - Summary: lowercase, imperative mood, concise

## A4. Analyze Staged Changes

Review the diff from A1. Determine whether all staged changes belong to a single feature or task.

- **Single feature/task** → continue to A5.
- **Multiple features/tasks** → list which files belong to which group. Inform the user that changes should be split into separate commits. **STOP — do not create a mixed commit.**

## A5. Draft Commit Message

Draft a commit message applying the convention from A3:

- Type must match the nature of the change (`feat` = new feature, `fix` = bug fix, `refactor` = restructuring, `chore` = maintenance).
- Summary must accurately describe "what changed" in the convention's style.
- Follow all aspects detected in A3 (scope, separator, casing, tense, length).

**Do not ask the user for confirmation unless explicitly instructed to do so. Proceed directly to A6.**

## A6. Create Commit

```bash
git commit -m "<message>"
```

- If commit **succeeds** → continue to A7.
- If commit **fails** (e.g. pre-commit hook) → show the error output, diagnose the cause, and **STOP.**

## A7. Report

```bash
git log --oneline -1
```

Show the output. Confirm the commit was created successfully.

# Input

No arguments. This command always follows the full decision tree.
