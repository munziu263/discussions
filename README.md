# Discussions

A lightweight decision record system for AI-assisted codebases. Track decisions through discussions (back-and-forth conversations) and experiments (hypothesis-driven implementation), with typed cross-references between them and to commits.

A lightweight alternative to GitHub Issues for private codebases.

## Install

```bash
npx skills@latest add munziu263/discussions
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

## Commands

| Command | Description |
|---------|-------------|
| `/discussion new <topic>` | Create a new discussion on a topic |
| `/discussion respond <file>` | Read and respond to latest user entries |
| `/discussion close <file>` | Close with a resolution summary |
| `/discussion experiment <file>` | Promote to a formal experiment |
| `/discussion list [all]` | List discussions (open by default) |
| `/discussion search <query>` | Full-text search across all records |
| `/discussion check` | Validate refs, find broken links, stale items |

## Setup

After installing, create per-project config at `.claude/decisions.local.md`:

```yaml
---
discussions_dir: discussions
experiments_dir: experiments
speakers:
  users: [your-name]
  ai: claude
---
```

Create the directories:

```bash
mkdir -p discussions experiments
```

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

## License

MIT
