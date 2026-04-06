Create a GitHub release for an existing git tag.

# Context

- Rule: **1 release = 1 tag. No exceptions.**
- Rule: **Follow every step of the decision tree in order. Do not skip or reorder steps unless the user explicitly instructs otherwise.**

# Decision Tree

```
START
│
├─ [A0] Check input
│  ├─ User provided tag name → save tag → [A2]
│  └─ No input → need to detect tag → [A1]
│
├─ [A1] Detect latest tag
│  │  Run: git describe --tags --abbrev=0
│  ├─ Tag found → save tag → [A2]
│  └─ No tags exist → inform user → STOP
│
├─ [A2] Validate tag
│  │  Run: git tag -l "<tag>"
│  ├─ Tag exists locally → [A3]
│  └─ Tag does not exist → inform user, list available tags → STOP
│
├─ [A3] Detect previous tag
│  │  Run: git tag --sort=-creatordate | head -10
│  │  Find the tag immediately before target tag
│  ├─ Previous tag found → save as comparison base → [A4]
│  └─ No previous tag → use initial commit as base → [A4]
│
├─ [A4] Gather changes between tags
│  │  Run: git log <prev-tag>..<tag> --oneline
│  │  Run: git diff <prev-tag>..<tag> --stat
│  └─ Save outputs → [A5]
│
├─ [A5] Check for existing release
│  │  Run: gh release view <tag>
│  ├─ Release exists → inform user, ask whether to overwrite → STOP or [A7]
│  └─ No release → [A6]
│
├─ [A6] Fetch existing releases
│  │  Run: gh release list --limit 5
│  │  Run: gh release view <latest> (if exists)
│  └─ Save outputs → [A7]
│
├─ [A7] Determine release convention
│  │  Analyze releases from A6
│  ├─ ≥ 3 releases → check for consistent pattern
│  │  ├─ Consistent convention detected → save detected convention → [A8]
│  │  └─ No consistent convention → save default convention → [A8]
│  └─ < 3 releases OR skipped A6 → save default convention → [A8]
│
├─ [A8] Categorize commits
│  │  Analyze commits from A4
│  │  Group by type: Breaking Changes, Features, Bug Fixes, Improvements, Chores
│  └─ Save categorized list → [A9]
│
├─ [A9] Draft release title & body
│  └─ Show draft to user → [A10]
│
├─ [A10] Confirm with user
│  ├─ User approves → [A11]
│  └─ User requests changes → revise draft → [A10]
│
├─ [A11] Create release
│  │  Run: gh release create <tag> --title "<title>" --notes "<body>"
│  ├─ OK → [A12]
│  └─ FAIL → show error → STOP
│
└─ [A12] Report
     Show release URL returned by gh release create
     Confirm release created successfully → END
```

# Actions

## A0. Check Input

Determine whether the user provided a specific tag name.

- If **user provided tag name** (e.g. `v1.3.0`) → save it as the target tag. Skip to A2.
- If **no input** → need to detect the latest tag. Continue to A1.

## A1. Detect Latest Tag

```bash
git describe --tags --abbrev=0
```

Read the output.

- If a **tag is found** → save it as the target tag. Continue to A2.
- If **no tags exist** → inform the user that no tags were found. **STOP.**

## A2. Validate Tag

```bash
git tag -l "<tag>"
```

Read the output.

- If the **tag exists locally** → continue to A3.
- If the **tag does not exist** → inform the user. List available tags with `git tag --sort=-creatordate | head -10`. **STOP.**

## A3. Detect Previous Tag

```bash
git tag --sort=-creatordate
```

Read the output. Find the tag immediately before the target tag in chronological order.

- If a **previous tag is found** → save it as the comparison base. Continue to A4.
- If **no previous tag** (target is the first tag) → use the initial commit as the comparison base. Continue to A4.

## A4. Gather Changes Between Tags

Run both commands and read their output:

```bash
git log <prev-tag>..<tag> --oneline
git diff <prev-tag>..<tag> --stat
```

This covers:
- **Commit history** between the two tags (`git log`)
- **File change summary** between the two tags (`git diff --stat`)

Save all outputs. These are the foundation for categorizing changes in A8 and drafting the release in A9.

## A5. Check for Existing Release

```bash
gh release view <tag>
```

- If a **release already exists** for this tag → inform the user. Ask whether to overwrite. If yes, proceed to A7 (will use `--latest` flag to update). If no, **STOP.**
- If **no release exists** → continue to A6.

## A6. Fetch Existing Releases

```bash
gh release list --limit 5
```

If releases exist, also fetch the latest release body for convention detection:

```bash
gh release view <latest-tag>
```

Read and save the outputs. These are used in A7 to detect convention.

## A7. Determine Release Convention

Analyze the releases from A6. Check the following aspects for a consistent pattern:

**Title:**
- **Format**: tag name only (`v1.2.0`) vs descriptive (`v1.2.0 — SSR Improvements`) vs no-v prefix (`1.2.0`)
- **Separator**: what separates version from description (` — `, ` - `, `: `, or none)

**Body:**
- **Language**: Korean, English, or mixed
- **Section structure**: what sections exist (e.g. `What's Changed`, `Bug Fixes`, `Features`, `Breaking Changes`)
- **Section headers**: format of headers (e.g. `## Bug Fixes`, `### Features`, `**Breaking Changes**`)
- **List style**: bullet points (`-`) vs `*` vs prose
- **Changelog link**: whether a `Full Changelog` comparison link is included at the bottom

Decision:

- **≥ 3 releases, consistent convention detected** → extract the full pattern. Save it for A9.
- **≥ 3 releases, no consistent convention** OR **< 3 releases** → save the default convention for A9:
  - Title: tag name only (e.g. `v1.3.0`)
  - Body language: English
  - Body structure:
    ```
    ## What's Changed

    ### Breaking Changes
    - item (only if applicable)

    ### Features
    - item (only if applicable)

    ### Bug Fixes
    - item

    ### Improvements
    - item

    **Full Changelog**: https://github.com/<owner>/<repo>/compare/<prev-tag>...<tag>
    ```
  - List style: bullet points with `-`
  - Omit empty sections

## A8. Categorize Commits

Analyze the commits from A4. Assign each commit to a category based on its prefix or content:

| Commit prefix / content | Category |
|---|---|
| `feat:`, `feature:` | Features |
| `fix:` | Bug Fixes |
| `refactor:`, `perf:`, `chore:`, `docs:`, `style:`, `test:` | Improvements |
| Contains "BREAKING", "breaking change", or `!:` | Breaking Changes |

Rules:
- A commit can appear in **Breaking Changes** AND another category if it is both breaking and a feature/fix.
- If a commit has no recognizable prefix, infer from its content.
- Group related commits into a single bullet when they address the same concern.

Save the categorized list for A9.

## A9. Draft Release Title & Body

Apply the convention determined in A7. Base the content on the categorized commits from A8 and the change stats from A4.

**Title:**
- Follow the detected or default convention.

**Body:**
- Follow the structure and language from the convention.
- Only include sections that have items (omit empty sections).
- Each bullet should describe the change from a **consumer's perspective** — what changed for them, not internal implementation details.
- Include the Full Changelog comparison link at the bottom.

Show the drafted title and body to the user.

## A10. Confirm with User

Wait for the user's response.

- If **user approves** → continue to A11.
- If **user requests changes** → revise the draft and show again. Repeat until approved.

## A11. Create Release

```bash
gh release create <tag> --title "<title>" --notes "<body>"
```

- If release creation **succeeds** → continue to A12.
- If release creation **fails** → show error output and **STOP.**

## A12. Report

Show the release URL returned by `gh release create`. Confirm the release was created successfully.

# Input

`$ARGUMENTS` — optional tag name (e.g. `v1.3.0`). If provided, use it as the target tag (skip A1). If not provided, detect the latest tag automatically.
