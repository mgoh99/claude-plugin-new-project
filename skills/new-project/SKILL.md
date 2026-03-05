---
name: new-project
description: Initialize a new Flask + Supabase + Vercel web application project from scratch. Use when starting a brand new project.
argument-hint: [app-description]
user-invocable: true
---

You are starting a new web application project. The user will describe what the app should do via `$ARGUMENTS`. If no arguments are provided, ask the user to describe their application requirements (what it does, key features, pages, user flows, database tables, etc.).

## Tech Stack

| Layer                  | Technology                                     |
| ---------------------- | ---------------------------------------------- |
| **Frontend**           | Flask templates (Jinja2) with HTML / CSS / JS  |
| **Backend**            | Python / Flask                                 |
| **Database**           | Supabase (PostgreSQL + Supabase Python client) |
| **Hosting/Deployment** | Vercel (via vercel-python runtime)             |

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

1. Initialize the full project structure
2. Set up the Flask app with Vercel-compatible config
3. Connect Supabase and create any needed tables (provide SQL migrations)
4. Build out the routes, templates, and logic per the user's application requirements
5. Make sure `vercel dev` works locally for testing
6. Initialize a git repository
7. Provide a clear README with setup and deployment steps

## Application Requirements

$ARGUMENTS
