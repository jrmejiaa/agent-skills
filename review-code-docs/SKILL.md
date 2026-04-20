---
name: review-code-docs
description: Iteratively find and fix in-code documentation issues in a project using partitioned auditors and a validator agent. Use this skill when the user asks to review, fix, or improve code documentation for a project. Accepts one argument ($0) which is the project path or alias.
---

This skill audits and fixes in-code documentation issues (comments, docstrings) in a project
by partitioning the codebase across multiple auditor agents, validating the results to remove
false positives, then applying fixes — iterating until no issues remain, no progress is made,
or a maximum of 3 iterations is reached.

## When to Use

- User asks to "fix docs", "review documentation", "improve docs", or "review code docs" for a project.

## Arguments

- `$0` — Project path.

## Agent Assumptions

- All tools are functional and will work without error. Do not test tools or make exploratory calls.
- Only call a tool if it is required to complete the task.

## Steps

Execute the following loop. Each pass through the loop is called an **iteration**.

### Step 1 — Gather Project Context

1. Resolve `$0` to its source directory. Read the project's `AGENTS.md` and `README.md` at the project root.
2. From `AGENTS.md` and `README.md`, follow any references to documentation style rules, conventions, or
    guidelines files. Read all referenced files. These, together with `AGENTS.md` and `README.md`, form the
    **project context**.
3. **Prerequisite check**: If neither `AGENTS.md` nor `README.md` exists, or neither references documentation
    style rules (directly or via a referenced file), stop immediately and print:

    > ⚠️ No documentation style rules found. Ensure your project's `AGENTS.md` or `README.md` references a
    > documentation style guide before running this skill.

4. Concatenate all gathered content into a single **project context** block. This block is passed to every
    subagent in subsequent steps.
5. Derive the list of source directories to scan from the project context (e.g., a "Directory Map" section in
    `AGENTS.md`). Also derive the list of files and directories to ignore (e.g., auto-generated files, test
    directories, mock directories) from the project context. If no directory map is found, fall back to scanning
    the project root with sensible default ignores (e.g., `node_modules`, `.git`, `build`, `dist`, `target`,
    `vendor`, `__pycache__`).
6. Only collect source code files (by extension). Do not include Markdown, prose, or configuration files.

### Step 2 — Partition

1. Collect all source files from the directories identified in Step 1, excluding ignored files and directories.
2. For C/C++ projects, group files by header + implementation pairs: if both `foo.hpp` and `foo.cpp` exist, they
    form a single unit and must land in the same partition. For all other languages, treat each file as its own
    unit.
3. Compute the number of partitions: `min(4, ceil(file_count / 12))`, where `file_count` is the number of file
    units (a header+implementation pair counts as 1).
4. Distribute file units across partitions as evenly as possible.

### Step 3 — Audit (parallel)

1. Launch one `opus-agent` subagent **per partition**, all in parallel. Each agent receives:
    - Its assigned file list (absolute paths).
    - The full **project context** block.
    - The following instruction:

      > Read all source files in your assigned file list. Read the provided project context
      > and extract every documentation rule it defines. Use those rules as your checklist.
      >
      > For each file in your assigned list, check every symbol (class, struct, method,
      > function, constant, typedef, member attribute, constructor — regardless of
      > visibility) against every rule in the checklist.
      >
      > Only flag issues that meet one of these criteria:
      >
      > 1. Missing documentation required by a rule in the project context.
      > 2. Factually incorrect documentation (describes behavior that contradicts the code).
      > 3. An explicit violation of a rule stated in the project context.
      >
      > **Do NOT flag:**
      > - Documentation that already matches the style guide examples in the project context.
      > - Stylistic rewording preferences (e.g. "could be clearer", "consider rephrasing").
      > - Anything not covered by a specific rule in the project context.
      >
      > For each issue found, produce a Markdown table row with these columns:
      >
      > | Issue # | File | Symbol | Severity | Problem |
      >
      > - **Issue #** — sequential number starting at 1.
      > - **File** — path relative to the project root.
      > - **Symbol** — the class, function, method, variable, or section heading affected.
      >   Use `(file-level)` when the issue is about the file itself.
      > - **Severity** — `low`, `medium`, or `high`.
      > - **Problem** — one-sentence description of what is wrong or missing.
      >
      > Return **only** the table. No preamble, no summary.

2. Collect all result tables and merge them into a single **raw issue list**.
    Renumber issues sequentially.

### Step 4 — Validate

1. Launch **one** `opus-agent` subagent with:
    - The full **project context** block.
    - The raw issue list from Step 3.
    - The absolute path to the project source directory.
    - The following instruction:

      > You are a documentation review validator. You receive a list of reported
      > documentation issues and the project's documentation rules (project context).
      >
      > For each issue in the list:
      > 1. Read the referenced source file and locate the symbol.
      > 2. Determine whether the issue is a genuine violation of a rule defined in the
      >    project context.
      > 3. Remove any issue that is a false positive — i.e. the documentation is already
      >    correct, the rule does not apply, or the issue contradicts guidance in the
      >    project context.
      >
      > Return a **validated issue table** with the same columns, containing only the
      > issues that are genuine violations. Renumber sequentially. If all issues are
      > false positives, return an empty table.
      >
      > Return **only** the table. No preamble, no summary.

2. The output is the **validated issue list**.

### Step 5 — Termination Check

- If the validated issue list is **empty**:

  > ✅ No documentation issues remain. Done after N iteration(s).

  and **stop**.

- If this is iteration **3**:

  > ⚠️ Max iterations reached. The following issues remain unresolved:

  Print the validated issue list and **stop**.

- If the validated issue count did not decrease compared to the previous iteration:

  > ⚠️ No progress detected between iterations. The following issues remain unresolved:

  Print the validated issue list and **stop**.

- Otherwise, print the validated issue list prefixed with the iteration number
  (e.g., `## Iteration 1 — Validated Issues`) and continue to Step 6.

### Step 6 — Fix (parallel)

1. Launch one `opus-agent` subagent **per partition** (same partitions as Step 2),
    all in parallel. Each agent receives:
    - Its assigned file list (absolute paths).
    - The full **project context** block.
    - The subset of the validated issue list that applies to its partition's files.
    - The following instruction:

      > Read all source files in your assigned file list. Fix every documentation issue
      > listed below. Edit the files in place. Follow the project's documentation style
      > as defined in the provided project context — match the golden examples exactly.
      > Do not add unrelated changes. Do not remove or modify code logic — only
      > documentation (comments, docstrings).
      >
      > After all fixes are applied, return a summary table with columns:
      >
      > | Issue # | File | Symbol | Action Taken |

2. Collect all fix summary tables and merge them.
3. Print the merged fix summary table to the user.

### Step 7 — Loop

Go back to **Step 3** and start the next iteration.
