# andy-plugins

Andy's Claude Code plugins marketplace.

## Installation

```bash
claude plugin install git@github.com:andyxhadji/andy-plugins.git
```

## Available Plugins

### auto-andy

Automated MR comment triage and response using middleman, kata, and roborev.

```bash
claude plugin install git@github.com:andyxhadji/andy-plugins.git --name auto-andy
```

**Skills:**
- `/auto-andy:triage` - Query middleman for new MR comments and create kata tasks
- `/auto-andy:address --auto` - Process tasks, auto-fix high-confidence ones, create specs for complex ones
- `/auto-andy:post-comments` - Post approved responses to MRs

See [auto-andy/CLAUDE.md](auto-andy/CLAUDE.md) for full documentation.
