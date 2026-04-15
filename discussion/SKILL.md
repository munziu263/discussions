---
name: discussion
description: Manage decision records (discussions and experiments). Create, respond, close, promote, search, and validate cross-references. Invoked with /discussion <action> [args].
---

# Discussion -- Decision Record Manager

Manage the decision record system: discussions (conversation threads exploring
a problem) and experiments (hypothesis-driven implementation records). Create
new discussions, respond to user comments, close resolved discussions, promote
discussions to formal experiments, search across all records, and validate
cross-references.

ARGUMENTS: The first word after `/discussion` is the action. Everything after
the action is the argument.

## Configuration

Before any action, read the per-project config from `.claude/decisions.local.md`
(YAML frontmatter). Extract:

- `discussions_dir` -- directory for discussion files (default: `discussions`)
- `experiments_dir` -- directory for experiment files (default: `experiments`)
- `speakers.users` -- list of human participant names (default: `[user]`)
- `speakers.ai` -- name for AI entries (default: `claude`)

If the file does not exist, use the defaults above.

Templates are bundled with this skill at [templates/discussion.md](templates/discussion.md)
and [templates/experiment.md](templates/experiment.md) (relative to this SKILL.md file).

## Usage

```
/discussion new <topic>           Create a new discussion on <topic>
/discussion respond <file>        Read the file and respond to new user entries
/discussion close <file>          Close a discussion with a resolution summary
/discussion experiment <file>     Create an experiment from a discussion
/discussion list [all]            List discussions (open by default)
/discussion search <query>        Full-text search across all records
/discussion check                 Validate refs, find broken links, stale items
```

The `<file>` argument can be:
- A full filename: `2026-02-10-performance-gap.md`
- A partial match: `performance-gap` or `performance`
- A date prefix: `2026-02-10` (lists all from that date)

## Data Model

### Discussion frontmatter

```yaml
---
title: "Topic Name"
status: open           # open | resolved | abandoned
priority: medium       # high | medium | low
opened: YYYY-MM-DD
closed:
tags: []
refs: []               # typed references (see below)
---
```

### Experiment frontmatter

```yaml
---
title: "Experiment Name"
status: open           # open | resolved | abandoned
opened: YYYY-MM-DD
closed:
tags: []
refs: []               # typed references (see below)
decision:              # adopt | adapt | abandon
---
```

### The refs field (used in both types)

```yaml
refs:
  - type: discussion
    path: discussions/2026-02-19-wizard-correlation.md
  - type: experiment
    path: experiments/034-infrastructure-consolidation.md
  - type: commit
    ref: r494
    vcs: svn
  - type: commit
    ref: 716324c
    vcs: git
```

The `refs` field is a list of typed references. Each entry has a `type` key.
For `discussion` and `experiment` types, use `path` (relative to repo root).
For `commit` types, use `ref` (hash or revision) and `vcs` (`git` or `svn`).

### Discussion body structure

The body contains:
- `# Title` heading
- `## Context` section explaining what prompted the discussion
- `## Key Questions` section
- Conversation entries separated by `---`, each headed `### speaker -- YYYY-MM-DD`
- `## Resolution` section (filled when closing)

### Experiment body structure

The body contains:
- `# Experiment NNN: Name` heading
- `## Hypothesis` section
- `## Approach` section
- `## Results` section
- `## Learnings` section
- `## Decision` section
- `## Next` section

## Actions

### /discussion new <topic>

1. Read config from `.claude/decisions.local.md`.
2. Read the discussion template from `templates/discussion.md` (relative to this skill file).
3. Determine today's date (YYYY-MM-DD format).
4. Generate a kebab-case slug from the topic.
5. Create `<discussions_dir>/YYYY-MM-DD-<slug>.md` using the template.
6. Fill in the frontmatter: title from topic, status `open`, priority `medium`,
   today's date, empty tags and refs.
7. Replace placeholder speaker names in the template (`### claude`, `### user`)
   with the configured speaker names (`speakers.ai`, first name in `speakers.users`).
8. Ask the user what the context and key questions are, OR if they provided
   enough detail, fill those in and write the opening `### <speakers.ai>` entry
   with initial analysis.
9. Use the Explore agent or research tools to investigate the topic before
   writing the opening entry. The opening entry should be substantive --
   include findings from reading relevant source code, experiments, and
   reference code.

### /discussion respond <file>

1. Read config from `.claude/decisions.local.md`.
2. Resolve the file (see file resolution below) within `<discussions_dir>`.
3. Read the full discussion file.
4. Find the most recent entry from any speaker listed in `speakers.users` --
   this is what to respond to. Any `### <name>` entry where `<name>` matches
   a name in the `speakers.users` list counts as a user entry.
5. Research the points raised: read source code, check experiments, check
   reference code, search the web if needed. Be thorough.
6. Append a new `### <speakers.ai> -- YYYY-MM-DD` entry after a `---` separator.
7. Address each point the user(s) raised. Be specific -- reference file paths,
   line numbers, experiment numbers.
8. If the discussion seems resolved, suggest changing status to `resolved`
   and explain why.
9. If new questions emerged, add them to the Key Questions section.

### /discussion close <file>

1. Read config from `.claude/decisions.local.md`.
2. Resolve the file within `<discussions_dir>`.
3. Read the full discussion file.
4. Summarize the conversation into the `## Resolution` section:
   - **Decision:** What was decided
   - **Next:** What it leads to (experiment ref, implementation, or "no action needed")
5. Update frontmatter:
   - Set `status: resolved` (or `abandoned` if the discussion was abandoned)
   - Set `closed: YYYY-MM-DD`
   - Add any relevant refs (e.g., to experiments that resulted from this discussion)
6. Add a final `### <speakers.ai> -- YYYY-MM-DD` entry summarizing the closure.

### /discussion experiment <file>

1. Read config from `.claude/decisions.local.md`.
2. Resolve the file within `<discussions_dir>`.
3. Read the full discussion file.
4. Confirm with the user that the discussion is ready for an experiment, unless
   the user has already indicated this clearly.
5. Read the experiment template from `templates/experiment.md` (relative to this skill file).
6. Determine the next experiment number by reading `<experiments_dir>/` and
   finding the highest existing numbered prefix (NNN-*.md). The new experiment
   number is max + 1, zero-padded to 3 digits.
7. Generate a kebab-case slug from the discussion topic or a more specific
   experiment name if appropriate.
8. Create `<experiments_dir>/NNN-<slug>.md` using the template.
9. Fill in the experiment:
   - **title** in frontmatter: derived from the discussion topic.
   - **Hypothesis:** Derived from the discussion's key questions and
     proposed direction.
   - **Approach:** Concrete steps, derived from the discussion entries.
   - **Status:** `open`
   - **refs:** Add a reference back to the source discussion:
     ```yaml
     refs:
       - type: discussion
         path: <discussions_dir>/YYYY-MM-DD-<slug>.md
     ```
10. **Bidirectional refs:** Update the discussion file to add a ref to the
    newly created experiment:
    - Add to the discussion's `refs` list:
      ```yaml
      - type: experiment
        path: <experiments_dir>/NNN-<slug>.md
      ```
11. Add a final `### <speakers.ai> -- YYYY-MM-DD` entry to the discussion
    noting the experiment was created, with the experiment path.
12. Report the experiment file path and number to the user.

### /discussion list [all]

1. Read config from `.claude/decisions.local.md`.
2. Read all files in `<discussions_dir>/` (exclude template files like
   000-template.md).
3. Parse YAML frontmatter from each file.
4. If `all` is NOT specified, filter to `status: open` only.
5. Display as a table:

```
OPEN DISCUSSIONS
================
Priority  Status    Date        Topic
--------  ------    ----        -----
high      open      2026-02-10  Developer Experience: Debugging
high      open      2026-02-10  Deployment: v2 Backport Strategy
medium    open      2026-02-10  Performance Gap (16x)
low       open      2026-02-10  Pydantic Dependency
```

6. Sort by priority (high > medium > low), then by date (newest first).
7. Show count: "N open, M resolved, K abandoned, J total"

### /discussion search <query>

Search across both `<discussions_dir>` and `<experiments_dir>` for a query
string (case-insensitive).

1. Read config from `.claude/decisions.local.md`.
2. Search for the query in:
   - Filenames (the slug portion)
   - Titles (from YAML frontmatter)
   - Tags (from YAML frontmatter)
   - Body text (full content)
3. Use the Grep tool for body text search. Use Glob + Read for frontmatter
   and filename matching.
4. Display results grouped by type, discussions first, then experiments:

```
SEARCH RESULTS: "correlation"
=============================

Discussions:
  [open]      2026-02-19  Wizard Correlation Value Discrepancy
  [resolved]  2026-02-17  Rosetta Retempt Data Migration

Experiments:
  [open]      043  Wizard Pipeline Decoupling
  [resolved]  037  Native RS Pairs

4 results found (2 discussions, 2 experiments)
```

5. Show status in brackets, date for discussions, number for experiments.
6. If no results found, say so clearly.

### /discussion check

Validate cross-references and report issues across both directories.

1. Read config from `.claude/decisions.local.md`.
2. Read all files in `<discussions_dir>/` and `<experiments_dir>/`.
3. Parse YAML frontmatter from each file, extracting `refs`, `status`,
   `opened`, and `closed`.
4. Perform the following checks:

**Broken references:**
- For each ref with `type: discussion` or `type: experiment`, verify the
  `path` points to an existing file. Report any broken paths.

**One-way references:**
- For each file A that refs file B (via `type: discussion` or
  `type: experiment`), check whether B also refs A. Report cases where
  A references B but B does not reference A.

**Stale discussions:**
- Report open discussions older than 14 days (based on `opened` date)
  with no recent activity. "Recent activity" means a conversation entry
  dated within the last 14 days. Check for `### ... -- YYYY-MM-DD` entries
  and compare the most recent date to today.

**Open experiments:**
- Report experiments with `status: open` as in-progress work that may
  need attention.

5. Display results grouped by check type:

```
REFERENCE CHECK
===============

Broken references:
  discussions/2026-02-10-example.md refs experiments/099-nonexistent.md (NOT FOUND)

One-way references:
  experiments/043-pipeline.md refs discussions/2026-02-20-pipeline.md (no back-ref)

Stale discussions (open > 14 days, no recent activity):
  discussions/2026-02-10-performance-gap.md (opened 2026-02-10, last activity 2026-02-12)

Open experiments:
  experiments/043-wizard-pipeline-decoupling.md (opened 2026-02-20)

Summary: 1 broken ref, 1 one-way ref, 1 stale discussion, 1 open experiment
```

6. If all checks pass, say: "All references valid. No issues found."

## File Resolution

When the user provides a `<file>` argument:

1. Check if it's an exact filename in `<discussions_dir>/`.
2. If not, search for files containing the argument as a substring
   (case-insensitive) in the filename.
3. If multiple matches, list them and ask the user to be more specific.
4. If no matches, report that no discussion was found and offer to create one.

## Conventions

- Speaker names: Use `speakers.users` (list) and `speakers.ai` from config.
  Do NOT hardcode names. Any name in the `speakers.users` list is a valid
  human participant. Multiple users can write entries in the same discussion.
- Dates: Always YYYY-MM-DD format.
- Use today's actual date, not a hardcoded one.
- Entries are separated by a single `---` (horizontal rule).
- Each entry starts with `### speaker -- YYYY-MM-DD`.
- Never modify or remove existing entries -- only append new ones.
- When responding, address the user's points directly and specifically.
  Reference file paths, line numbers, experiment numbers.
- Keep the frontmatter up to date: tags, refs, status.
- Statuses are limited to: `open`, `resolved`, `abandoned`. No other statuses.
- Templates are at `templates/discussion.md` and `templates/experiment.md`
  (relative to this skill file).
- The discussions directory and experiments directory are configured in
  `.claude/decisions.local.md` and default to `discussions/` and
  `experiments/` at the repo root.

## Important

- NEVER delete or modify existing conversation entries. Only append.
- ALWAYS read the full discussion before responding.
- ALWAYS read config from `.claude/decisions.local.md` before any action.
- When creating experiments, use the template from `templates/experiment.md`
  (relative to this skill file).
- When creating experiments from discussions, ALWAYS create bidirectional refs
  (experiment refs discussion AND discussion refs experiment).
- All file paths in responses should be relative to the repo root or absolute.
- The refs field replaces the old `leads-to` field. Do not use `leads-to`.
