---
name: review-docs
description: Iteratively find and fix documentation issues in a project using partitioned auditors and a validator agent. Use this skill when the user asks to review, fix, or improve documentation for a project. Accepts one argument ($0) which is the project path or alias.
---

This skill audits and fixes documentation issues in a project by partitioning the codebase
across multiple auditor agents, validating the results to remove false positives, then
applying fixes — iterating until no issues remain or a maximum of 3 iterations is reached.

## When to Use

- User asks to "fix docs", "review documentation", "improve docs", or "make docs" for a project.

## Arguments

- `$0` — Project path.

## Agent Assumptions

- All tools are functional and will work without error. Do not test tools or make exploratory calls.
- Only call a tool if it is required to complete the task.

## Steps

Execute the following loop. Each pass through the loop is called an **iteration**.

### Step 1 — Gather Project Context

1. Resolve `$0` to its source directory. Read the project's `AGENTS.md` (check both
   `sources/<project>/AGENTS.md` and `sources/<project>/.kiro/AGENTS.md`).
2. Read all files in the project's `.kiro/steering/` directory, if it exists. These are
   the **steering files** — they contain the project's authoritative rules for documentation
   style, conventions, and what must not be flagged.
3. **Prerequisite check**: If no steering file defines documentation style rules, stop
   immediately and print:

   > ⚠️ No documentation style rules found in `.kiro/steering/`. Create a
   > `documentation-style.md` with your project's documentation rules before running
   > this skill.

4. Concatenate the `AGENTS.md` content and all steering file contents into a single
   **project context** block. This block is passed to every subagent in subsequent steps.
5. Derive the list of source directories to scan from the AGENTS.md (e.g. its
   "Directory Map" section). Also derive the list of files and directories to ignore
   (e.g. auto-generated files, test directories, mock directories, integration test
   directories) from the AGENTS.md "Files to Ignore" section and any other relevant
   guidance in the project context.

### Step 2 — Partition

1. Collect all source files from the directories identified in Step 1, excluding
   ignored files and directories.
2. Group files by header + implementation pairs: if both `foo.hpp` and `foo.cpp` exist,
   they form a single unit and must land in the same partition.
3. Compute the number of partitions: `min(4, ceil(file_count / 12))`, where
   `file_count` is the number of file units (a header+implementation pair counts as 1).
4. Distribute file units across partitions as evenly as possible.

### Step 3 — Audit (parallel)

1. Launch one `opus-agent` subagent **per partition**, all in parallel. Each agent receives:
    - Its assigned file list (absolute paths).
    - The full **project context** block (AGENTS.md + all steering files).
    - The following instruction:

      > Read all source files in your assigned file list. Read the provided steering files
      > and extract every documentation rule they define. Use those rules as your checklist.
      >
      > For each file in your assigned list, check every public and protected symbol
      > (class, struct, method, function, constant, typedef, member attribute, constructor)
      > against every rule in the checklist.
      >
      > Only flag issues that meet one of these criteria:
      >
      > 1. Missing documentation required by a rule in the steering files.
      > 2. Factually incorrect documentation (describes behavior that contradicts the code).
      > 3. An explicit violation of a rule stated in the steering files or AGENTS.md.
      >
      > **Do NOT flag:**
      > - Documentation that already matches the style guide examples in the steering files.
      > - Stylistic rewording preferences (e.g. "could be clearer", "consider rephrasing").
      > - Anything not covered by a specific rule in the steering files.
      > - Destructors.
      > - File-level descriptions.
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
    - The full **project context** block (AGENTS.md + all steering files).
    - The raw issue list from Step 3.
    - The absolute path to the project source directory.
    - The following instruction:

      > You are a documentation review validator. You receive a list of reported
      > documentation issues and the project's documentation rules (steering files).
      >
      > For each issue in the list:
      > 1. Read the referenced source file and locate the symbol.
      > 2. Determine whether the issue is a genuine violation of a rule defined in the
      >    steering files.
      > 3. Remove any issue that is a false positive — i.e. the documentation is already
      >    correct, the rule does not apply, or the issue contradicts guidance in the
      >    steering files or AGENTS.md.
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

- Otherwise, print the validated issue list prefixed with the iteration number
  (e.g., `## Iteration 1 — Validated Issues`) and continue to Step 6.

### Step 6 — Fix (parallel)

1. Launch one `opus-agent` subagent **per partition** (same partitions as Step 2),
   all in parallel. Each agent receives:
    - Its assigned file list (absolute paths).
    - The full **project context** block (AGENTS.md + all steering files).
    - The subset of the validated issue list that applies to its partition's files.
    - The following instruction:

      > Read all source files in your assigned file list. Fix every documentation issue
      > listed below. Edit the files in place. Follow the project's documentation style
      > as defined in the provided steering files — match the golden examples exactly.
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
