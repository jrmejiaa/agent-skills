---
name: pr-summary
description: Create PR summary for a project. Use this skill when the user asks to create a pull request summary of a new feature on a project.
---

This skill guides creation of a PR summary for a new feature of a project.

## When to Use
- User says "create pull request", "write pr summary", "create pr summary", "pr description", "summarize my branch"

## Steps to create the PR summary

In order to create a PR summary you MUST follow precisely the following steps:

1. Get the number of commits that separates the current branch from the base branch (typically `main` or `master`). If the branch has no commits ahead, inform the user and stop.
2. Get the log of the last `NUM_COMMITS` commits, where `NUM_COMMITS` is the number found in step 1.
3. Use the log from step 2 as input for the creation of the Pull Request summary.
4. Propose a PR name that captures most (if not all) changes from the git log.
5. Every commit message may contain issue/ticket ID references (e.g. `JIRA-123`, `#42`). All unique IDs must appear as bullet points in the footer of the PR summary.
6. Do not include intermediate or work-in-progress changes that were superseded by later commits.
7. Start the PR name with a level-three Markdown header (`###`).
8. All following section headers must be level-four (`####`).
9. The structure of the PR summary must be:
    - **General Overview** — no more than 200 characters.
    - `---`
    - **Key Changes** — subdivided by area. Each subdivision must be a bold bullet point.
    - `---`
    - **Limitations** — only if explicitly mentioned in the git log. Omit the section otherwise.
    - `---`
    - **Refactoring** — only if refactoring is present in the log. Omit the section otherwise.
    - `---`
    - **Breaking Changes** — only if you identify breaking changes in the log. Use your judgement and re-think before including anything here. Omit the section otherwise.
    - `---`
    - **Unique IDs** - only if you identify unique ids on step 5. Use a bullet list of all unique ids found.
10. For terms that reference specific code identifiers, use inline code (`` `word` ``) to represent them.
11. Output the result as a raw Markdown code block so it is not rendered by the CLI.
12. Do not mention commit hashes or any abbreviation of them in the final output.
