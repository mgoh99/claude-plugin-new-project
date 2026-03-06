---
name: add-auth
description: Add Supabase GoTrue authentication with Microsoft OAuth (Azure SSO) to any Flask project. Creates login page, auth module, password reset, and all supporting routes.
argument-hint: [app-name]
user-invocable: true
---

You are adding authentication to an existing Flask project. The user may provide an app name via `$ARGUMENTS` — use it for branding on the login page. If no arguments, look at the project's existing Flask app to determine the app name, or ask.

## What This Creates

A complete auth system matching the clear_view pattern:

1. **`auth.py`** — Supabase GoTrue REST auth module (no SDK)
2. **Login page template** — Email/password + Microsoft OAuth button
3. **Auth callback template** — Handles PKCE and recovery token flows client-side
4. **Password reset template** — Set new password form
5. **Access denied template** — Shown when user lacks permission
6. **403 template** — Forbidden error page
7. **Auth CSS** — Login page styles (appended to existing stylesheet)
8. **Auth routes** — Added to the Flask app
9. **Config updates** — Auth-related environment variables

## Architecture

- **No Supabase SDK** — direct REST calls with `requests` library and `apikey` header
- **JWT verification** — local HS256 verification using `SUPABASE_JWT_SECRET`
- **Token refresh** — automatic refresh on expiry without requiring re-login
- **Azure SSO** — PKCE flow: `/auth/azure` → Azure redirect → `/auth/callback` → code exchange → session
- **Password recovery** — Supabase emails magic link → `/auth/callback` extracts fragment → `/reset-password`
- **Session** — Flask signed cookie stores `access_token`, `refresh_token`, `user_email`, `user_id`

## Step-by-Step Instructions

### 1. Create `auth.py`

Create the auth module with these exact functions:

```python
"""
Supabase Auth module.
Handles login, token verification, refresh, and the @login_required decorator.
Uses Supabase GoTrue REST API (no SDK).
"""

import functools
import jwt
import requests
from flask import session, redirect, url_for, request, render_template, jsonify
import config


class AuthError(Exception):
    """Raised when authentication fails."""
    pass


def login_user(email, password):
    """Authenticate with Supabase GoTrue via POST /auth/v1/token?grant_type=password."""
    url = f"{config.SUPABASE_URL.rstrip('/')}/auth/v1/token?grant_type=password"
    headers = {
        "apikey": config.SUPABASE_API_KEY,
        "Content-Type": "application/json",
    }
    payload = {"email": email, "password": password}
    resp = requests.post(url, json=payload, headers=headers, timeout=10)

    if resp.status_code != 200:
        content_type = resp.headers.get("content-type", "")
        if "application/json" in content_type:
            error_body = resp.json()
            msg = error_body.get("error_description") or error_body.get("msg") or "Login failed"
        else:
            msg = "Login failed"
        raise AuthError(msg)

    data = resp.json()
    return {
        "access_token": data["access_token"],
        "refresh_token": data["refresh_token"],
        "expires_at": data.get("expires_at", 0),
        "user_email": data.get("user", {}).get("email", email),
        "user_id": data.get("user", {}).get("id", ""),
    }


def verify_token(access_token):
    """Locally verify a Supabase JWT using the JWT secret (HS256)."""
    return jwt.decode(
        access_token,
        config.SUPABASE_JWT_SECRET,
        algorithms=["HS256"],
        audience="authenticated",
    )


def refresh_access_token(refresh_token):
    """Refresh an expired access token via POST /auth/v1/token?grant_type=refresh_token."""
    url = f"{config.SUPABASE_URL.rstrip('/')}/auth/v1/token?grant_type=refresh_token"
    headers = {
        "apikey": config.SUPABASE_API_KEY,
        "Content-Type": "application/json",
    }
    payload = {"refresh_token": refresh_token}
    resp = requests.post(url, json=payload, headers=headers, timeout=10)

    if resp.status_code != 200:
        raise AuthError("Session expired. Please log in again.")

    data = resp.json()
    return {
        "access_token": data["access_token"],
        "refresh_token": data["refresh_token"],
        "expires_at": data.get("expires_at", 0),
    }


def update_user_password(access_token, new_password):
    """Update a user's password using their access token via PUT /auth/v1/user."""
    url = f"{config.SUPABASE_URL.rstrip('/')}/auth/v1/user"
    headers = {
        "apikey": config.SUPABASE_API_KEY,
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json",
    }
    payload = {"password": new_password}
    resp = requests.put(url, json=payload, headers=headers, timeout=10)

    if resp.status_code != 200:
        content_type = resp.headers.get("content-type", "")
        if "application/json" in content_type:
            error_body = resp.json()
            msg = error_body.get("error_description") or error_body.get("msg") or "Password update failed"
        else:
            msg = "Password update failed"
        raise AuthError(msg)

    return resp.json()


def get_current_user():
    """Return the current user dict from the Flask session, or None."""
    if "user_email" in session and "access_token" in session:
        return {
            "email": session["user_email"],
            "user_id": session.get("user_id", ""),
        }
    return None


def login_required(f):
    """
    Decorator that protects a route. Checks Flask session for a valid JWT.
    If the token is expired, attempts a refresh. If refresh fails, redirects to /login.
    """
    @functools.wraps(f)
    def decorated(*args, **kwargs):
        access_token = session.get("access_token")
        refresh_token_val = session.get("refresh_token")

        if not access_token:
            session["next_url"] = request.url
            return redirect(url_for("login_page"))

        try:
            verify_token(access_token)
        except jwt.ExpiredSignatureError:
            if refresh_token_val:
                try:
                    new_tokens = refresh_access_token(refresh_token_val)
                    session["access_token"] = new_tokens["access_token"]
                    session["refresh_token"] = new_tokens["refresh_token"]
                except AuthError:
                    session.clear()
                    session["next_url"] = request.url
                    return redirect(url_for("login_page"))
            else:
                session.clear()
                session["next_url"] = request.url
                return redirect(url_for("login_page"))
        except jwt.InvalidTokenError:
            session.clear()
            session["next_url"] = request.url
            return redirect(url_for("login_page"))

        return f(*args, **kwargs)

    return decorated
```

### 2. Create Login Template

Create `templates/login.html`. Use the app name from `$ARGUMENTS` for the brand text. The template must include:

- Microsoft OAuth button linking to `/auth/azure`
- "or" divider
- Email/password form posting to `/login`
- Success and error alert display
- Script to catch Supabase recovery redirects that arrive at the root with `#access_token=...`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Sign In — APP_NAME</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}" />
</head>
<body class="login-body">
  <div class="login-container">
    <div class="login-card">
      <div class="login-brand">APP_NAME</div>
      <p class="login-subtitle">Sign in to continue</p>

      {% if success %}
      <div class="alert alert-success" style="margin-bottom: 16px;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/>
          <polyline points="22 4 12 14.01 9 11.01"/>
        </svg>
        {{ success }}
      </div>
      {% endif %}

      {% if error %}
      <div class="alert alert-warning" style="margin-bottom: 16px;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/>
          <line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/>
        </svg>
        {{ error }}
      </div>
      {% endif %}

      <a href="{{ url_for('auth_azure') }}" class="login-btn-microsoft">
        <svg width="16" height="16" viewBox="0 0 21 21" xmlns="http://www.w3.org/2000/svg">
          <rect x="1" y="1" width="9" height="9" fill="#f25022"/>
          <rect x="11" y="1" width="9" height="9" fill="#7fba00"/>
          <rect x="1" y="11" width="9" height="9" fill="#00a4ef"/>
          <rect x="11" y="11" width="9" height="9" fill="#ffb900"/>
        </svg>
        Sign in with Microsoft
      </a>

      <div class="login-divider"><span>or</span></div>

      <form method="POST" action="{{ url_for('login_page') }}">
        <div class="login-field">
          <label for="email">Email</label>
          <input type="email" id="email" name="email" required autofocus placeholder="you@company.com" />
        </div>
        <div class="login-field">
          <label for="password">Password</label>
          <input type="password" id="password" name="password" required placeholder="Enter your password" />
        </div>
        <button type="submit" class="login-btn">Sign In</button>
      </form>
    </div>
  </div>
  <script>
    (function () {
      var hash = window.location.hash.substring(1);
      if (!hash) return;
      var params = new URLSearchParams(hash);
      var accessToken = params.get('access_token');
      if (accessToken) {
        window.location.href = '/auth/callback' + window.location.hash;
      }
    })();
  </script>
</body>
</html>
```

Replace `APP_NAME` with the actual app name.

### 3. Create Auth Callback Template

Create `templates/auth_callback.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Redirecting... — APP_NAME</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}" />
</head>
<body class="login-body">
  <div class="login-container">
    <div class="login-card" style="text-align: center;">
      <div class="login-brand">APP_NAME</div>
      <p class="login-subtitle" id="status-msg">Processing authentication...</p>
    </div>
  </div>
  <script>
    (function () {
      var query = new URLSearchParams(window.location.search);
      var hash = window.location.hash.substring(1);
      var fragment = hash ? new URLSearchParams(hash) : new URLSearchParams();

      var code = query.get('code') || fragment.get('code');
      var accessToken = fragment.get('access_token');
      var refreshToken = fragment.get('refresh_token') || '';
      var type = fragment.get('type') || '';

      var error = query.get('error_description') || query.get('error')
                || fragment.get('error_description') || fragment.get('error');

      if (error) {
        window.location.href = '/login?error=' + encodeURIComponent(error);
        return;
      }

      if (code) {
        document.getElementById('status-msg').textContent = 'Signing you in...';
        fetch('/auth/exchange-code', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ code: code }),
        })
          .then(function (resp) { return resp.json(); })
          .then(function (data) {
            if (data.redirect) {
              window.location.href = data.redirect;
            } else {
              window.location.href = '/login?error=' + encodeURIComponent(data.error || 'Sign in failed');
            }
          })
          .catch(function () {
            window.location.href = '/login?error=' + encodeURIComponent('Sign in failed. Please try again.');
          });
        return;
      }

      if (accessToken) {
        if (type === 'recovery') {
          window.location.href = '/reset-password?access_token=' + encodeURIComponent(accessToken)
            + '&refresh_token=' + encodeURIComponent(refreshToken);
        } else {
          document.getElementById('status-msg').textContent = 'Signing you in...';
          fetch('/auth/token-login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ access_token: accessToken, refresh_token: refreshToken }),
          })
            .then(function (resp) { return resp.json(); })
            .then(function (data) {
              if (data.success) {
                window.location.href = data.redirect || '/';
              } else {
                window.location.href = '/login?error=' + encodeURIComponent(data.error || 'Sign in failed');
              }
            })
            .catch(function () {
              window.location.href = '/login?error=' + encodeURIComponent('Sign in failed. Please try again.');
            });
        }
        return;
      }

      document.getElementById('status-msg').textContent = 'No authentication data found. Redirecting to login...';
      setTimeout(function () { window.location.href = '/login'; }, 1500);
    })();
  </script>
</body>
</html>
```

### 4. Create Password Reset Template

Create `templates/reset_password.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Set Password — APP_NAME</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}" />
</head>
<body class="login-body">
  <div class="login-container">
    <div class="login-card">
      <div class="login-brand">APP_NAME</div>
      <p class="login-subtitle">Set your new password</p>

      {% if error %}
      <div class="alert alert-warning" style="margin-bottom: 16px;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/>
          <line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/>
        </svg>
        {{ error }}
      </div>
      {% endif %}

      <form method="POST" action="{{ url_for('reset_password_page') }}">
        <input type="hidden" name="access_token" value="{{ access_token }}" />
        <input type="hidden" name="refresh_token" value="{{ refresh_token }}" />
        <div class="login-field">
          <label for="password">New Password</label>
          <input type="password" id="password" name="password" required autofocus
                 placeholder="At least 6 characters" minlength="6" />
        </div>
        <div class="login-field">
          <label for="confirm_password">Confirm Password</label>
          <input type="password" id="confirm_password" name="confirm_password" required
                 placeholder="Re-enter your password" minlength="6" />
        </div>
        <button type="submit" class="login-btn">Set Password</button>
      </form>
    </div>
  </div>
</body>
</html>
```

### 5. Create Access Denied Template

Create `templates/access_denied.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Access Denied — APP_NAME</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}" />
</head>
<body class="login-body">
  <div class="login-container">
    <div class="login-card">
      <div class="login-brand">APP_NAME</div>
      <p class="login-subtitle">Access Denied</p>
      <div class="alert alert-warning" style="margin-bottom: 16px;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/>
          <line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/>
        </svg>
        Your account has not been approved for access. Please contact an administrator.
      </div>
      <a href="/logout" class="login-btn" style="display:block;text-align:center;text-decoration:none;">Sign Out</a>
    </div>
  </div>
</body>
</html>
```

### 6. Create 403 Template

Create `templates/403.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Forbidden — APP_NAME</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}" />
</head>
<body class="login-body">
  <div class="login-container">
    <div class="login-card">
      <div class="login-brand">APP_NAME</div>
      <p class="login-subtitle">Forbidden</p>
      <div class="alert alert-warning" style="margin-bottom: 16px;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/>
          <line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/>
        </svg>
        You do not have permission to view this page.
      </div>
      <a href="/" class="login-btn" style="display:block;text-align:center;text-decoration:none;">Return Home</a>
    </div>
  </div>
</body>
</html>
```

### 7. Add Auth CSS

Append the following CSS to the project's main stylesheet (typically `static/css/style.css`). If the file doesn't exist, create it with these styles:

```css
/* ── Auth Pages ── */
.alert {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 6px;
  font-size: 13px;
}
.alert-warning {
  background: #fffbeb;
  border: 1px solid #fde68a;
  color: #92400e;
}
.alert-success {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  color: #166534;
}
.login-body {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  background: #f8fafc;
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}
.login-container {
  width: 100%;
  max-width: 380px;
  padding: 24px;
}
.login-card {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 32px;
}
.login-brand {
  font-size: 14px;
  font-weight: 600;
  letter-spacing: -0.01em;
  margin-bottom: 4px;
}
.login-subtitle {
  font-size: 13px;
  color: #64748b;
  margin-bottom: 24px;
}
.login-field {
  margin-bottom: 16px;
}
.login-field label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  margin-bottom: 4px;
  color: #334155;
}
.login-field input {
  width: 100%;
  padding: 8px 12px;
  font-size: 13px;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  outline: none;
  transition: border-color 0.15s;
  font-family: inherit;
  color: #0f172a;
  background: #fff;
  box-sizing: border-box;
}
.login-field input:focus {
  border-color: #94a3b8;
}
.login-field input::placeholder {
  color: #94a3b8;
}
.login-btn {
  width: 100%;
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 500;
  background: #1e293b;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  margin-top: 8px;
  font-family: inherit;
  transition: background 0.15s;
}
.login-btn:hover {
  background: #0f172a;
}
.login-btn-microsoft {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  width: 100%;
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 500;
  background: #fff;
  color: #1e293b;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.15s, border-color 0.15s;
  text-decoration: none;
  box-sizing: border-box;
}
.login-btn-microsoft:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}
.login-divider {
  display: flex;
  align-items: center;
  margin: 16px 0;
  gap: 12px;
}
.login-divider::before,
.login-divider::after {
  content: "";
  flex: 1;
  height: 1px;
  background: #e2e8f0;
}
.login-divider span {
  font-size: 12px;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
```

### 8. Add Auth Routes to Flask App

Add these imports and routes to the Flask app file. Adapt the route function names and redirect targets to match the project's existing routes:

```python
import hashlib
import secrets
import base64
from datetime import timedelta
import requests as http_requests
from flask import Flask, render_template, jsonify, request, session, redirect, url_for
from auth import login_required, login_user, get_current_user, update_user_password, AuthError
import config

app = Flask(__name__)
app.secret_key = config.FLASK_SECRET_KEY
app.permanent_session_lifetime = timedelta(seconds=config.SESSION_LIFETIME_SECONDS)


# ── Auth Routes ──

@app.route("/login", methods=["GET", "POST"])
def login_page():
    if request.method == "GET":
        if get_current_user():
            return redirect(url_for("index"))  # Change to your home route
        error = request.args.get("error")
        return render_template("login.html", error=error)

    email = request.form.get("email", "").strip()
    password = request.form.get("password", "")

    if not email or not password:
        return render_template("login.html", error="Email and password are required.")

    try:
        tokens = login_user(email, password)
        session.permanent = True
        session["access_token"] = tokens["access_token"]
        session["refresh_token"] = tokens["refresh_token"]
        session["user_email"] = tokens["user_email"]
        session["user_id"] = tokens["user_id"]

        next_url = session.pop("next_url", None) or url_for("index")  # Change to your home route
        return redirect(next_url)
    except AuthError as e:
        return render_template("login.html", error=str(e))


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login_page"))


@app.route("/auth/azure")
def auth_azure():
    """Initiate Azure OAuth via Supabase using PKCE flow."""
    code_verifier = secrets.token_urlsafe(64)
    digest = hashlib.sha256(code_verifier.encode("ascii")).digest()
    code_challenge = base64.urlsafe_b64encode(digest).rstrip(b"=").decode("ascii")

    session["pkce_code_verifier"] = code_verifier

    site_url = request.url_root.rstrip("/")
    redirect_to = f"{site_url}/auth/callback"
    supabase_url = config.SUPABASE_URL.rstrip("/")
    oauth_url = (
        f"{supabase_url}/auth/v1/authorize"
        f"?provider=azure"
        f"&redirect_to={redirect_to}"
        f"&code_challenge={code_challenge}"
        f"&code_challenge_method=s256"
    )
    return redirect(oauth_url)


@app.route("/auth/callback")
def auth_callback():
    """Handle Supabase auth redirects (PKCE code or fragment-based tokens)."""
    code = request.args.get("code")
    if code:
        code_verifier = session.pop("pkce_code_verifier", "")
        if not code_verifier:
            return redirect(url_for("login_page", error="Session expired. Please try again."))

        supabase_url = config.SUPABASE_URL.rstrip("/")
        resp = http_requests.post(
            f"{supabase_url}/auth/v1/token?grant_type=pkce",
            json={"auth_code": code, "code_verifier": code_verifier},
            headers={
                "apikey": config.SUPABASE_API_KEY,
                "Content-Type": "application/json",
            },
            timeout=10,
        )

        if resp.status_code != 200:
            error_msg = "Sign in failed"
            try:
                body = resp.json()
                error_msg = body.get("error_description") or body.get("msg") or error_msg
            except Exception:
                pass
            return redirect(url_for("login_page", error=error_msg))

        data = resp.json()
        user = data.get("user", {})
        session.permanent = True
        session["access_token"] = data["access_token"]
        session["refresh_token"] = data.get("refresh_token", "")
        session["user_email"] = user.get("email", "")
        session["user_id"] = user.get("id", "")

        next_url = session.pop("next_url", None) or url_for("index")  # Change to your home route
        return redirect(next_url)

    return render_template("auth_callback.html")


@app.route("/auth/exchange-code", methods=["POST"])
def auth_exchange_code():
    """Exchange a PKCE auth code for tokens (called by auth_callback.html JS)."""
    data = request.get_json(force=True)
    code = data.get("code", "")
    if not code:
        return jsonify({"error": "No auth code provided"}), 400

    code_verifier = session.pop("pkce_code_verifier", "")
    if not code_verifier:
        return jsonify({"error": "Session expired. Please try again."}), 400

    supabase_url = config.SUPABASE_URL.rstrip("/")
    resp = http_requests.post(
        f"{supabase_url}/auth/v1/token?grant_type=pkce",
        json={"auth_code": code, "code_verifier": code_verifier},
        headers={
            "apikey": config.SUPABASE_API_KEY,
            "Content-Type": "application/json",
        },
        timeout=10,
    )

    if resp.status_code != 200:
        error_msg = "Sign in failed"
        try:
            body = resp.json()
            error_msg = body.get("error_description") or body.get("msg") or error_msg
        except Exception:
            pass
        return jsonify({"error": error_msg}), 401

    tokens = resp.json()
    user = tokens.get("user", {})
    session.permanent = True
    session["access_token"] = tokens["access_token"]
    session["refresh_token"] = tokens.get("refresh_token", "")
    session["user_email"] = user.get("email", "")
    session["user_id"] = user.get("id", "")

    next_url = session.pop("next_url", None) or url_for("index")  # Change to your home route
    return jsonify({"redirect": next_url})


@app.route("/auth/token-login", methods=["POST"])
def auth_token_login():
    """Accept tokens directly (magic link / implicit flow) and set up the session."""
    from auth import verify_token

    data = request.get_json(force=True)
    access_token = data.get("access_token", "")
    refresh_token_val = data.get("refresh_token", "")

    if not access_token:
        return jsonify({"error": "No access token provided"}), 400

    try:
        decoded = verify_token(access_token)
        session.permanent = True
        session["access_token"] = access_token
        session["refresh_token"] = refresh_token_val
        session["user_email"] = decoded.get("email", "")
        session["user_id"] = decoded.get("sub", "")

        next_url = session.pop("next_url", None) or url_for("index")  # Change to your home route
        return jsonify({"success": True, "redirect": next_url})
    except Exception as e:
        return jsonify({"error": str(e)}), 401


@app.route("/reset-password", methods=["GET", "POST"])
def reset_password_page():
    if request.method == "GET":
        access_token = request.args.get("access_token")
        refresh_token_val = request.args.get("refresh_token")
        if not access_token:
            return redirect(url_for("login_page"))
        return render_template(
            "reset_password.html",
            access_token=access_token,
            refresh_token=refresh_token_val or "",
        )

    access_token = request.form.get("access_token", "")
    refresh_token_val = request.form.get("refresh_token", "")
    new_password = request.form.get("password", "")
    confirm_password = request.form.get("confirm_password", "")

    if not new_password or len(new_password) < 6:
        return render_template(
            "reset_password.html",
            access_token=access_token,
            refresh_token=refresh_token_val,
            error="Password must be at least 6 characters.",
        )

    if new_password != confirm_password:
        return render_template(
            "reset_password.html",
            access_token=access_token,
            refresh_token=refresh_token_val,
            error="Passwords do not match.",
        )

    try:
        update_user_password(access_token, new_password)
        return render_template(
            "login.html",
            error=None,
            success="Password updated. Sign in with your new password.",
        )
    except AuthError as e:
        return render_template(
            "reset_password.html",
            access_token=access_token,
            refresh_token=refresh_token_val,
            error=str(e),
        )


@app.route("/access-denied")
def access_denied_page():
    return render_template("access_denied.html")
```

**Important**: Change all `url_for("index")` references to match the project's actual home page route name.

### 9. Update Config

Ensure `config.py` includes these auth-related variables:

```python
import os
from dotenv import load_dotenv

load_dotenv()

SUPABASE_URL = os.getenv("SUPABASE_URL", "")
SUPABASE_API_KEY = os.getenv("SUPABASE_API_KEY", "")
SUPABASE_JWT_SECRET = os.getenv("SUPABASE_JWT_SECRET", "")
FLASK_SECRET_KEY = os.getenv("FLASK_SECRET_KEY", "change-me-in-production")
SESSION_LIFETIME_SECONDS = int(os.getenv("SESSION_LIFETIME_SECONDS", "86400"))
```

### 10. Update .env.example

Add these variables to `.env.example`:

```
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_API_KEY=your-anon-key
SUPABASE_JWT_SECRET=your-jwt-secret
FLASK_SECRET_KEY=your-random-secret-key
SESSION_LIFETIME_SECONDS=86400
```

### 11. Update requirements.txt

Ensure these packages are listed:

```
flask
requests
python-dotenv
PyJWT
```

### 12. Protect Existing Routes

Add `@login_required` decorator to all routes that should require authentication:

```python
@app.route("/")
@login_required
def index():
    ...
```

## Supabase Setup Requirements

Tell the user they need to configure in their Supabase project:

1. **Authentication > Providers > Azure**: Enable and configure with Azure AD app registration (Client ID, Client Secret, Tenant URL)
2. **Authentication > URL Configuration**: Add the app's callback URL (`https://your-domain.com/auth/callback`) to Redirect URLs
3. **Authentication > Email Templates**: Customize the password recovery email template if desired
4. The JWT secret is found in Supabase Dashboard > Project Settings > API > JWT Secret

## Notes

- This auth system does NOT use the Supabase Python SDK — it makes direct REST calls
- The PKCE flow is more secure than implicit flow for OAuth
- Tokens are stored in Flask's signed session cookie (server-side secret)
- The `@login_required` decorator handles token refresh transparently
- The `role_required` decorator from clear_view is NOT included by default — add it if you need role-based access control
