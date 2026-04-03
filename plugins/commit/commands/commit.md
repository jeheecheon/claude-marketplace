Split changes into separate commits by feature unit (one feature/task per commit).

Steps:

1. Run `git status` and `git diff --staged` to verify staged changes
2. Run `git log --oneline -10` to review existing commit message conventions
3. Draft a commit message that accurately reflects the changes and follows project style
4. Create the commit with the drafted message

Commit message format:

- `[type]: [short summary]`
- Types: `feat`, `chore`, `refactor`, `fix`
- Summary: lowercase, imperative mood, concise

Rules:

- NEVER include `Co-Authored-By` or any AI attribution
- Leave no trace of AI assistance in commits (no AI-related tags, footers, or metadata)
