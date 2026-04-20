---
name: commit
description: Create a git commit for the current diff of the project. Use this skill when the user asks to commit, create a git message, or commit changes.
---

This skill guides creation of a git commit message for the current project.

## When to Use

- User asks to create, write, or generate a git commit message.

## Out of Scope

- Amending commits (`--amend`) is not handled by this skill.

## Steps

1. Extract the ticket/issue ID from the current branch name (e.g.,
    `jira/JRMA-3666/feature-integrate-epoll-to-mainline` → `JRMA-3666`). If no recognizable ID is found, skip the
    `id:` footer silently — do not ask the user. If multiple IDs are detected in the branch name, stop with an error
    explaining that only one ticket ID per branch is allowed.
2. Check for staged files. Do not assume you have the latest information. Use diff to make sure in every iteration
    of the skill.
3. If both staged and unstaged diffs are empty and there are no untracked files, report "nothing to commit" and stop.
4. If files are already staged, use only those as the basis for the message. Otherwise, use the full working tree
    diff and include untracked files (via `git ls-files --others --exclude-standard`) in the analysis.
5. For large diffs (20+ files changed), use `git diff --stat` first to get an overview, then selectively read the
    most relevant file diffs to generate the message.
6. Read the git diff output and generate a commit message following the Conventional Commits guidelines in
    `references/commits-guidelines.md`.
7. Choose an appropriate commit type (`feat`, `fix`, `refactor`, etc.) based on what the diff contains.
8. Infer a scope from the diff when the change is clearly localized to a single module or area (e.g.,
    `feat(parser): ...`). Omit the scope when the change spans multiple areas. Keep scopes to one word.
9. Keep the title line under 72 characters. Wrap body lines at 72 characters; break long bullet items with
    continuation indentation.
10. Include a commit body when the diff touches more than one file or when the change isn't self-evident from the
    subject line alone. Skip the body for trivial, single-file changes where the subject fully describes the intent.
11. For breaking changes, add both the `BREAKING CHANGE:` footer and the `!` symbol after the type in the title.
12. In the footer, reference the ID extracted in step 1 using the format `id: <extracted-id>` (not `Refs`).
13. If the diff contains changes that would require different commit types (e.g., a `feat` and an unrelated `fix`),
    explain your reasoning and propose splitting into separate commits. Present all proposed messages together in a
    numbered list.
14. For split commits, save the full diff to a temp file under `/tmp/` before staging. Use `git add -p` or
    `git add --patch` to handle intra-file splits when needed. After all split commits are done, verify that the
    combined committed diff matches the original saved diff.
15. Present the proposed commit message(s) and ask the user to confirm (c), edit (e), or reject (r) before
    proceeding. Wait for the user's response.
    - On **(c)onfirm**: stage and commit. For multi-commit splits, stage and commit each one sequentially.
    - On **(e)dit**: write the message to a temp file under `/tmp/`, open it with `vi`, and proceed to step 16.
    - On **(r)eject**: stop.
16. After the user edits the file, validate the edited message against conventional commit rules (has a valid type,
    subject under 72 chars, proper footer format). If validation fails, warn the user and ask them to fix it or
    confirm they want to commit as-is.
17. If the edited message passes validation (or the user forces it), compare the saved content with the originally
    proposed message. If anything changed, treat the edited version as confirmed and commit immediately — do NOT ask
    for confirmation again. If nothing changed, wait for the user to confirm (c) or reject (r).
18. If the commit fails due to a pre-commit hook, report the hook output to the user and stop. Do not retry
    automatically.
