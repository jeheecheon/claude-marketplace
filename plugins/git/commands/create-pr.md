# Create Pull Request

Create a PR for the current branch using the project template.

## Steps

1. If on `develop` or `main`, create a feature branch: `git checkout -b feature/<short-kebab-description>`
2. Run `git log develop..HEAD --oneline` and `git diff develop...HEAD --stat` to understand all changes
3. Push branch: `git push -u origin <branch>`
4. Read `.github/pull_request_template.md` and draft PR body following its format
5. Create PR:

```bash
gh pr create --title "<type>: <summary>" --body "<body>" --base develop
```

## Rules

- Title format: `<type>: <summary>` (feat, fix, refactor, chore)
- If an argument is provided, use it as the title
- Link related issues with `Closes #N` or `Refs #N`
- Keep WHY focused on the problem/motivation
- Keep WHAT focused on implementation decisions
- Write PR body (WHY/WHAT content) in Korean
- Infer `<type>` from branch name prefix (e.g., `feature/` → `feat`, `fix/` → `fix`, `refactor/` → `refactor`)
