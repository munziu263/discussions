# Discussions

A lightweight decision record system for AI-assisted codebases. Track decisions
through discussions (back-and-forth conversations) and experiments
(hypothesis-driven implementation), with typed cross-references between them
and to commits.

A lightweight alternative to GitHub Issues for private codebases.

## Install

Via the [skills CLI](https://github.com/mattpocock/skills):

```bash
npx skills@latest add munziu263/discussions
```

Or as a Claude Code plugin:

```bash
npx skills@latest add munziu263/discussions --copy
```

## The Workflow

```
  Idea or bug
       |
       v
  /discussion new "topic"        <-- open a conversation thread
       |
       v
  /discussion respond <file>     <-- claude researches and responds
       |                             (repeat as needed)
       v
  /discussion experiment <file>  <-- promote to a formal experiment
       |
       v
  /discussion close <file>       <-- record what was decided
```

Events can happen in any order. An experiment can come first. A discussion
can close without an experiment. Commits can be referenced from either type.
The `refs` field links everything together as a chain of related events.

## Setup

After installing, create per-project config at `.claude/decisions.local.md`:

```yaml
---
discussions_dir: discussions
experiments_dir: experiments
speakers:
  users: [your-name]          # add more: [alice, bob, carol]
  ai: claude
demos:
  tool: none                  # set to "tinker" to create interactive demos
---
```

Create the directories:

```bash
mkdir -p discussions experiments
```

All fields are optional -- sensible defaults are used if the config file
doesn't exist.

## Commands

### Start a new discussion

```
/discussion new why are correlation values different across pages
```

Creates `discussions/2026-02-23-correlation-value-discrepancy.md` with a
conversation thread. Claude researches the topic and writes the opening entry.

### Respond to a discussion

```
/discussion respond correlation
```

Partial filename matching -- finds the discussion, reads the latest user
entry, researches the points raised, and appends a response.

### Close a discussion

```
/discussion close correlation
```

Writes a Resolution section summarizing what was decided and what it leads to.

### Promote a discussion to an experiment

```
/discussion experiment correlation
```

Creates `experiments/044-correlation-fix.md` with hypothesis and approach
derived from the discussion. Links both files to each other. If `demos.tool`
is set to `tinker`, also creates an interactive demo.

### List open discussions

```
/discussion list
```

```
OPEN DISCUSSIONS
================
Priority  Status  Date        Topic
--------  ------  ----        -----
high      open    2026-02-20  Zero Expense Ratio in xFactor Results
medium    open    2026-02-16  Test Portfolio Design

2 open, 13 resolved, 0 abandoned, 15 total
```

### List all discussions (including resolved)

```
/discussion list all
```

### Search across everything

```
/discussion search wizard
```

Searches filenames, titles, tags, and body text across both discussions
and experiments. Answers the question: "what did we decide about X?"

### Validate cross-references

```
/discussion check
```

Reports broken file references, one-way links (A refs B but B doesn't ref A),
stale open discussions, and in-progress experiments.

## How Records Link Together

Both discussions and experiments use typed `refs` in YAML frontmatter:

```yaml
refs:
  - type: discussion
    path: discussions/2026-02-19-correlation.md
  - type: experiment
    path: experiments/037-native-rs-pairs.md
  - type: commit
    ref: 716324c
    vcs: git
```

References are bidirectional by convention -- `/discussion check` validates this.

## Record Types

### Discussions

Conversation threads exploring a problem. The body has Context, Key Questions,
back-and-forth entries (`### speaker -- date`), and a Resolution section.

```yaml
---
title: "Wizard Correlation Value Discrepancy"
status: open           # open | resolved | abandoned
priority: high         # high | medium | low
opened: 2026-02-19
closed:
tags: [wizard, correlation]
refs: []
---
```

### Experiments

Hypothesis-driven implementation records. The body has Hypothesis, Approach,
Results, Learnings, Decision, and Next sections.

```yaml
---
title: "Native RS Pair Computation"
status: resolved       # open | resolved | abandoned
opened: 2026-02-11
closed: 2026-02-11
tags: [performance, native]
refs:
  - type: discussion
    path: discussions/2026-02-10-native-extensions.md
decision: adopt        # adopt | adapt | abandon
---
```

## Optional: Interactive Demos with Tinker

When `demos.tool` is set to `tinker` in your config, the `/discussion experiment`
command creates an interactive demo alongside each experiment using
[tinker-cli](https://github.com/munziu263/tinker-cli).

Demos live at `.tinker/NNN-<slug>/` and include:
- `demo.py` -- percent-format Python script for step-by-step exploration
- `tinker.toml` -- manifest with batch replay (`tinker run`) and interactive
  REPL (`tinker repl`) support

```bash
# Replay the demo
uvx --from tinker-cli tinker run .tinker/044-correlation-fix/

# Interactive exploration
uvx --from tinker-cli tinker repl .tinker/044-correlation-fix/
```

Tinker supports python, c, zig, go, javascript, and bash. Set `demos.tool: none`
(the default) to skip demo creation entirely.

## License

MIT
