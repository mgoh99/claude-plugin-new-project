# New Project - Claude Code Plugin

A Claude Code skill that scaffolds a new **Flask + Supabase + Vercel** web application project from scratch.

## What It Does

When invoked, this skill instructs Claude to:

1. Create a full project structure (Flask app, templates, static files, Supabase client)
2. Configure Vercel deployment (`vercel.json`, serverless entry point)
3. Set up Supabase connection with environment variables
4. Build out routes, templates, and logic based on your app description
5. Provide SQL migrations for any needed database tables
6. Initialize a git repository
7. Generate a README with setup and deployment steps

## Tech Stack

| Layer              | Technology                                     |
| ------------------ | ---------------------------------------------- |
| **Frontend**       | Flask templates (Jinja2) with HTML / CSS / JS  |
| **Backend**        | Python / Flask                                 |
| **Database**       | Supabase (PostgreSQL + Supabase Python client) |
| **Hosting**        | Vercel (via vercel-python runtime)             |

## Installation

```bash
claude plugin add github:mgoh99/claude-plugin-new-project
```

## Usage

In Claude Code, type:

```
/new-project A todo app with user authentication and task categories
```

Or just `/new-project` with no arguments and Claude will ask you to describe your app.

## Manual Installation

If you prefer not to use the plugin system, copy the skill directly:

```bash
mkdir -p ~/.claude/skills/new-project
curl -o ~/.claude/skills/new-project/SKILL.md \
  https://raw.githubusercontent.com/mgoh99/claude-plugin-new-project/main/skills/new-project/SKILL.md
```

## License

MIT
