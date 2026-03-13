# Clearspace Tools

A Claude Code skill marketplace with plugins for scaffolding Flask web applications.

## Marketplace Structure

```
clearspace-tools/
├── plugins/
│   └── clearspace-code/          # Plugin: Flask project skills
│       ├── plugin.json
│       └── skills/
│           ├── new-project/      # Flask + Supabase + Vercel app
│           ├── new-local-project/# Flask + SQLite local app
│           └── add-auth/         # Supabase auth with Azure SSO
└── README.md
```

## Installation

### Install the full plugin

Clone the repo and copy the plugin's skills into your Claude Code config:

```bash
git clone https://github.com/mgoh99/clearspace-tools.git
cp -r clearspace-tools/plugins/clearspace-code/skills/* ~/.claude/skills/
```

### Install a single skill

You can install individual skills directly from GitHub without cloning:

**new-project** — Flask + Supabase + Vercel app scaffold:
```bash
mkdir -p ~/.claude/skills/new-project
curl -o ~/.claude/skills/new-project/SKILL.md \
  https://raw.githubusercontent.com/mgoh99/clearspace-tools/main/plugins/clearspace-code/skills/new-project/SKILL.md
```

**new-local-project** — Flask + SQLite local app scaffold:
```bash
mkdir -p ~/.claude/skills/new-local-project
curl -o ~/.claude/skills/new-local-project/SKILL.md \
  https://raw.githubusercontent.com/mgoh99/clearspace-tools/main/plugins/clearspace-code/skills/new-local-project/SKILL.md
```

**add-auth** — Supabase GoTrue auth with Microsoft OAuth:
```bash
mkdir -p ~/.claude/skills/add-auth
curl -o ~/.claude/skills/add-auth/SKILL.md \
  https://raw.githubusercontent.com/mgoh99/clearspace-tools/main/plugins/clearspace-code/skills/add-auth/SKILL.md
```

## Skills

### `/new-project`

Scaffolds a full **Flask + Supabase + Vercel** web application with:
- Flask backend with Jinja2 templates
- Supabase (PostgreSQL) database integration
- Vercel deployment configuration
- Tailwind CSS (CDN or CLI)
- Optional authentication (via `add-auth`)

```
/new-project A todo app with user authentication and task categories
```

### `/new-local-project`

Scaffolds a **Flask + SQLite** app for local development with:
- Flask backend with Jinja2 templates
- SQLite database (no cloud services)
- Tailwind CSS (CDN or CLI)

```
/new-local-project An inventory tracker for my home library
```

### `/add-auth`

Adds **Supabase GoTrue authentication** with Microsoft OAuth (Azure SSO) to any existing Flask project:
- Email/password login
- Microsoft SSO via PKCE flow
- Password reset
- `@login_required` decorator
- No Supabase SDK or JWT secret required

```
/add-auth MyAppName
```

## License

MIT
