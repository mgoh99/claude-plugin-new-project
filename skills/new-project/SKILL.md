---
name: new-project
description: Initialize a new Flask + Supabase + Vercel web application project from scratch. Use when starting a brand new project.
argument-hint: "app-description"
user-invocable: true
---

You are starting a new web application project. The user will describe what the app should do via `$ARGUMENTS`. If no arguments are provided, ask the user to describe their application requirements (what it does, key features, pages, user flows, database tables, etc.).

## Tech Stack

| Layer                  | Technology                                     |
| ---------------------- | ---------------------------------------------- |
| **Frontend**           | Flask templates (Jinja2) with HTML / CSS / JS + Tailwind CSS |
| **Backend**            | Python / Flask                                 |
| **Database**           | Supabase (PostgreSQL + Supabase Python client) |
| **Hosting/Deployment** | Vercel (via vercel-python runtime)             |

## Tailwind CSS Setup

Ask the user which Tailwind option they prefer, and suggest one based on the project type:

- **Option 1 — CDN (default):** Add `<script src="https://cdn.tailwindcss.com"></script>` to `base.html`. No build step required. Suggest this for prototypes, internal tools, simple apps, or when the user hasn't specified.
- **Option 2 — Tailwind CLI:** Install via `npm install -D tailwindcss`, configure `tailwind.config.js` to scan `templates/**/*.html`, and run the CLI to build into `static/css/style.css`. Add a `build:css` script to `package.json`. Suggest this for production apps, large projects, or apps with a custom design system where bundle size matters.

If the user does not specify, use **Option 1** and mention they can switch to Option 2 later for production.

## Authentication

After gathering application requirements, ask the user: **"Does this app need user authentication (login/logout)?"**

- If **yes**: after the project scaffold is complete, run the `add-auth` skill to add Supabase GoTrue authentication with Microsoft OAuth (Azure SSO). Pass the app name as the argument.
- If **no**: skip auth setup entirely.

## Project Structure

Create the following project structure in the current working directory:

```
project-name/
├── api/
│   └── index.py              # Vercel serverless entry point (Flask app)
├── templates/
│   ├── base.html             # Base layout template
│   └── index.html            # Home page
├── static/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── main.js
├── services/
│   └── supabase_client.py    # Supabase connection and helpers
├── requirements.txt
├── vercel.json               # Vercel deployment config
├── .env.example              # Environment variable template
├── .gitignore
└── README.md
```

## Configuration Details

### vercel.json
Route all requests through the Flask app in `api/index.py`. Use the `@vercel/python` runtime. Serve static files from `/static`.

### Supabase
Use the `supabase` Python library. Initialize the client using `SUPABASE_URL` and `SUPABASE_KEY` environment variables. Keep all DB logic in `services/supabase_client.py`.

### Flask
The Flask app must live in `api/index.py` so Vercel can pick it up as a serverless function. Use:

```python
Flask(__name__,
      template_folder='../templates',
      static_folder='../static')
```

### Environment Variables
- `SUPABASE_URL`
- `SUPABASE_KEY`
- `FLASK_SECRET_KEY`

## Instructions

1. Ask the user to describe their app (if not provided via `$ARGUMENTS`)
2. Ask which Tailwind option they prefer (suggest based on project type; default to Option 1)
3. Ask if the app needs user authentication
4. Initialize the full project structure
5. Set up the Flask app with Vercel-compatible config
6. Connect Supabase and create any needed tables (provide SQL migrations)
7. Set up Tailwind CSS per the chosen option
8. Build out the routes, templates, and logic per the user's application requirements
9. Set the Flask development server to launch on a random port in the **5050–5999** range (e.g. `port = random.randint(5050, 5999)`). Print the chosen port to the console on startup so the user knows where to find the app.
10. Use `python3` (not `python`) in all local run instructions, scripts, and README commands — the user is on macOS where `python3` is the correct command. Also use `pip3` instead of `pip` for package installation.
11. Make sure `vercel dev` works locally for testing
12. If auth was requested, run the `add-auth` skill now
13. Initialize a git repository
14. Provide a clear README with setup and deployment steps

## Application Requirements

$ARGUMENTS
