---
name: commit
description: Create a git commit for the current diff of the project. Use this skill when the user asks to commit, create a git message, or commit changes.
---

This skill guides creation of a git commit message for the current project.

## When to Use

- User asks to create, write, or generate a git commit message.

## Steps

1. Extract the ticket/issue ID from the current branch name (e.g.,
   `jira/JRMA-3666/feature-integrate-epoll-to-mainline` →
   `JRMA-3666`). If no recognizable ID is found, skip the `id:` footer
   silently — do not ask the user.
2. Check for staged files. Do not assume you have the latest
   information. Use diff to make sure in every iteration of the skill.
3. If both staged and unstaged diffs are empty, report "nothing to
   commit" and stop.
4. If files are already staged, use only those as the basis for the
   message. Otherwise, use the full working tree diff.
5. Read the git diff output and generate a commit message following the
   Conventional Commits guidelines in `references/commits-guidelines.md`.
6. Choose an appropriate commit type (`feat`, `fix`, `refactor`, etc.)
   based on what the diff contains.
7. Keep the title line under 72 characters. Wrap body lines at 72
   characters; break long bullet items with continuation indentation.
8. For breaking changes, add both the `BREAKING CHANGE:` footer and the
   `!` symbol after the type in the title.
9. In the footer, reference the ID extracted in step 1 using the format
   `id: <extracted-id>` (not `Refs`).
10. If the diff covers multiple unrelated changes, explain your reasoning
    and propose splitting into separate commits. Present all proposed
    messages together in a numbered list.
11. Present the proposed commit message(s) and ask the user to confirm
    (c), edit (e), or reject (r) before proceeding. Wait for the user's
    response.
    - On **(c)onfirm**: stage and commit. For multi-commit splits, stage
      and commit each one sequentially.
    - On **(e)dit**: write the message to a temp file under `/tmp/`, open
      it with `vi`, and proceed to step 12.
    - On **(r)eject**: stop.
12. After the user edits the file, compare the saved content with the
    originally proposed message. If anything changed, treat the edited
    version as confirmed and commit immediately — do NOT ask for
    confirmation again. If nothing changed, wait for the user to confirm
    (c) or reject (r).
