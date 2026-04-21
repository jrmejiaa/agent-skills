---
name: pr-summary
description: Create PR summary for a project. Use this skill when the user asks to create a pull request summary of a new feature on a project.
---

This skill guides creation of a PR summary for a new feature of a project.

## When to Use

- User says "create pull request", "write pr summary", "create pr summary", "pr description", "summarize my branch"

## Steps to create the PR summary

In order to create a PR summary you MUST follow precisely the following steps:

1. Determine the base branch by checking the repository's default branch (e.g., via `git remote show origin`). 
    If detection is ambiguous or fails, ask the user. Do not assume `main` or `master`.
2. Get the number of commits that separates the current branch from the base branch. If the branch has no commits
    ahead, inform the user and stop.
3. Get the full commit log (subject + body) of the last `NUM_COMMITS` commits, where `NUM_COMMITS` is the number
    found in step 2. Exclude merge commits (e.g., `--no-merges`).
4. Evaluate the quality of the commit messages. If commit messages are insufficient to produce a meaningful summary
    (e.g., `"fix"`, `"wip"`, `"asdf"`, `"temp"`, or empty bodies with vague subjects), STOP with a critical error
    explaining that the commit messages are too poor to generate a summary. Do not fall back to reading diffs.
5. Use the log from step 3 as input for the creation of the Pull Request summary.
6. Propose a PR title in plain text that captures most (if not all) changes from the git log. Output it separately
    from the PR body — do not include it as a heading inside the body.
7. Every commit message may contain issue/ticket ID references (e.g., `JIRA-123`, `#42`). Collect all unique IDs for
    the Unique IDs section.
8. Do not include intermediate or work-in-progress changes that were superseded by later commits.
9. If commits follow conventional commit format (e.g., `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`), use the
    prefixes to inform section categorization (e.g., `refactor:` → Refactoring, `feat:`/`fix:` → Key Changes).
10. For terms that reference specific code identifiers, use inline code (`` `word` ``) to represent them.
11. The structure of the PR body must be:
    - **General Overview** — no more than 280 characters.
    - `---`
    - **Unique IDs** — only if unique IDs were found in step 7. Use a bullet list of all unique IDs.
    - `---`
    - **Key Changes** — subdivided by area. Each subdivision must be a bold bullet point. Prefer grouping by domain
        concern (e.g., **Authentication**, **Billing**, **API**). Fall back to directory-based grouping if the commits
        don't have clear domain boundaries.
    - `---`
    - **Limitations** — only if explicitly mentioned in the git log. Omit the section otherwise.
    - `---`
    - **Refactoring** — only if refactoring is present in the log. Omit the section otherwise.
    - `---`
    - **Breaking Changes** — only if you identify breaking changes in the log. Use your judgement and re-think before
        including anything here. Omit the section otherwise.
12. Only place a `---` between sections that are actually present. Never output consecutive separators or a trailing
    separator.
13. All section headers must be level-four Markdown headers (`###`).
14. Output the result as a raw Markdown code block so it is not rendered by the CLI.
15. Do not mention commit hashes or any abbreviation of them in the final output.
16. After outputting the summary, copy it to the system clipboard. Auto-detect the Linux clipboard command:
    - Wayland: `wl-copy`
    - X11: `xclip -selection clipboard`
    - If neither is available, skip clipboard copy and inform the user the summary was printed but not copied.
