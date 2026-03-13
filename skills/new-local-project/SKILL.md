---
name: new-local-project
description: Initialize a new local Flask + SQLite + Tailwind CSS web application project. No cloud services — runs entirely on localhost.
argument-hint: "app-description"
user-invocable: true
---

You are starting a new local web application project. The user will describe what the app should do via `$ARGUMENTS`. If no arguments are provided, ask the user to describe their application requirements (what it does, key features, pages, user flows, database tables, etc.).

## Tech Stack

| Layer                  | Technology                                     |
| ---------------------- | ---------------------------------------------- |
| **Frontend**           | Flask templates (Jinja2) with HTML / CSS / JS + Tailwind CSS |
| **Backend**            | Python / Flask                                 |
| **Database**           | SQLite (via Python's built-in `sqlite3` module) |
| **Hosting/Deployment** | Local only (Flask development server)          |

## Tailwind CSS Setup

Ask the user which Tailwind option they prefer, and suggest one based on the project type:

- **Option 1 — CDN (default):** Add `<script src="https://cdn.tailwindcss.com"></script>` to `base.html`. No build step required. Suggest this for prototypes, internal tools, simple apps, or when the user hasn't specified.
- **Option 2 — Tailwind CLI:** Install via `npm install -D tailwindcss`, configure `tailwind.config.js` to scan `templates/**/*.html`, and run the CLI to build into `static/css/style.css`. Add a `build:css` script to `package.json`. Suggest this for production apps, large projects, or apps with a custom design system where bundle size matters.

If the user does not specify, use **Option 1** and mention they can switch to Option 2 later.

## Project Structure

Create the following project structure in the current working directory:

```
project-name/
├── app.py                       # Flask application entry point
├── templates/
│   ├── base.html                # Base layout template
│   └── index.html               # Home page
├── static/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── main.js
├── services/
│   └── database.py              # SQLite connection and helpers
├── instance/                    # SQLite database files (gitignored)
├── requirements.txt
├── .env.example                 # Environment variable template
├── .gitignore
└── README.md
```

## Configuration Details

### SQLite
Use Python's built-in `sqlite3` module. Store the database file at `instance/app.db`. Keep all DB logic in `services/database.py`, including:
- A `get_db()` function that returns a connection with `row_factory = sqlite3.Row`
- An `init_db()` function that creates tables on first run
- A `close_db()` function for teardown

Register `close_db` with `app.teardown_appcontext`. Call `init_db()` on app startup to ensure tables exist.

### Flask
The Flask app lives in `app.py` at the project root. Use:

```python
app = Flask(__name__)
app.config['SECRET_KEY'] = os.environ.get('FLASK_SECRET_KEY', 'dev-secret-key')
```

### Environment Variables
- `FLASK_SECRET_KEY`

## Instructions

1. Ask the user to describe their app (if not provided via `$ARGUMENTS`)
2. Ask which Tailwind option they prefer (suggest based on project type; default to Option 1)
3. Initialize the full project structure
4. Set up the Flask app with SQLite database
5. Create any needed tables in `services/database.py` with an `init_db()` function
6. Set up Tailwind CSS per the chosen option
7. Build out the routes, templates, and logic per the user's application requirements
8. Pick a single fixed port between **5500–5800** for this project and hardcode it. The port should not change between restarts.
9. Use `python3` (not `python`) in all local run instructions, scripts, and README commands — the user is on macOS where `python3` is the correct command. Also use `pip3` instead of `pip` for package installation.
10. Initialize a git repository
11. Provide a clear README with setup and run steps

## Application Requirements

$ARGUMENTS
