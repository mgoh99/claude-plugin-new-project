# Clearspace Tools

## Project
- Claude Code skill marketplace repo — GitHub: mgoh99/clearspace-tools
- Structure: `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`
- Plugin metadata: `plugins/<plugin-name>/plugin.json`

## Skills Installation
- Claude Code has NO `/plugin` command — do not suggest it
- Install skills by copying SKILL.md files to `~/.claude/skills/<skill-name>/`
- Example: `cp plugins/clearspace-code/skills/new-project/SKILL.md ~/.claude/skills/new-project/`

## Available Skills
- `new-project` — scaffold Flask + Supabase + Vercel app
- `new-local-project` — scaffold for local development
- `add-auth` — add authentication to existing project

## Commands
- `gh repo rename <name> --repo mgoh99/<old-name> --yes` — rename GitHub repo
