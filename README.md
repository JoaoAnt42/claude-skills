# claude-skills

Personal Claude Code slash commands for code review and repository health.

## Skills

| Command | Description |
|---------|-------------|
| `/code_review <PR-URL>` | Review a GitHub PR — presents findings for selection before posting inline comments |
| `/final_review [PR-URL\|task-file]` | Deep feature review before pushing — correctness, architecture, security |
| `/repo_doctor [branch\|--all]` | Full repository health audit — bugs, tech debt, security, saves report to `docs/` |

## Installation

Copy the skills you want into your Claude Code commands directory:

```bash
# Clone the repo
git clone https://github.com/JoaoAnt42/claude-skills.git

# Copy all skills
cp claude-skills/commands/*.md ~/.claude/commands/

# Or copy individual skills
cp claude-skills/commands/code_review.md ~/.claude/commands/
```

That's it. The commands are immediately available in Claude Code as `/code_review`, `/final_review`, and `/repo_doctor`.

## Requirements

- [Claude Code](https://claude.ai/code)
- [GitHub CLI (`gh`)](https://cli.github.com/) — authenticated with `gh auth login`
