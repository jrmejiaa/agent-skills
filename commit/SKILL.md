---
name: commit
description: Create a git commit for the current diff of the project. Use this skill when the user asks to commit, create a git message, or commit changes.
---

This skill guides creation of a git commit message for the current project.

## When to Use

- User asks to create, write, or generate a git commit message.

## Out of Scope

- Amending commits (`--amend`) is not handled by this skill.

## Hard Gates (check BEFORE generating any message)

1. **Staged-only rule**: If `git diff --staged` is non-empty, use ONLY the staged diff.
   Do NOT read, analyze, or reference unstaged changes or untracked files.
   Skip `git diff` (unstaged) and `git ls-files --others` entirely.

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
5. Collect the diff content:
    - For small diffs (<20 files): capture the full diff output.
    - For large diffs (20+ files): capture `git diff --stat` and the full diff.
6. **Delegate message generation to a subagent named `opus-agent`.** Write the diff content to `/tmp/commit-diff.txt`
    and invoke a subagent using `agent_name: "opus-agent"` with the following query:

    > Generate a conventional commit message for the diff in `/tmp/commit-diff.txt`.
    > Follow the guidelines in `/home/mejia/.kiro/skills/commit/references/commits-guidelines.md`.
    > The ticket ID is: `<extracted-id>` (or "none" if not found).
    > Rules:
    > - Base your message ONLY on what the diff shows. Do not invent, assume, or infer
    >   anything beyond what is visible in the added/removed lines.
    > - A line prefixed with `+` is new. A line prefixed with `-` is removed.
    >   If something only appears as `+` with no corresponding `-`, it is NEW, not
    >   renamed or moved.
    > - If something appears as both `-` (old location) and `+` (new location) with
    >   the same or similar content, it was MOVED or RENAMED.
    > - Choose an appropriate commit type (feat, fix, refactor, etc.).
    > - Infer a scope when the change is localized to one module. Omit when it spans
    >   multiple areas. Keep scopes to one word.
    > - Keep the title under 72 characters. Wrap body at 72 characters.
    > - Include a body when the diff touches more than one file or the change isn't
    >   self-evident from the subject alone.
    > - If the diff contains changes requiring different commit types, propose splitting
    >   into separate commits with reasoning.
    > - For the footer, use `id: <ticket-id>` format (not `Refs`). Skip if no ID.
    > - For breaking changes, use both `!` after type and `BREAKING CHANGE:` footer.
    > - Output ONLY the commit message text, nothing else.

7. Present the subagent's proposed commit message(s) to the user and ask to confirm (c), edit (e), or reject (r).
    Wait for the user's response.
    - On **(c)onfirm**: stage and commit. For multi-commit splits, stage and commit each one sequentially.
    - On **(e)dit**: write the message to a temp file under `/tmp/`, open it with `vi`, and proceed to step 8.
    - On **(r)eject**: stop.
8. After the user edits the file, validate the edited message against conventional commit rules (has a valid type,
    subject under 72 chars, proper footer format). If validation fails, warn the user and ask them to fix it or
    confirm they want to commit as-is.
9. If the edited message passes validation (or the user forces it), compare the saved content with the originally
    proposed message. If anything changed, treat the edited version as confirmed and commit immediately — do NOT ask
    for confirmation again. If nothing changed, wait for the user to confirm (c) or reject (r).
10. If the commit fails due to a pre-commit hook, report the hook output to the user and stop. Do not retry
    automatically.
11. For split commits, save the full diff to a temp file under `/tmp/` before staging. Use `git add -p` or
    `git add --patch` to handle intra-file splits when needed. After all split commits are done, verify that the
    combined committed diff matches the original saved diff.
