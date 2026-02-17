# Vercel deploy fixes (Feb 2026)

This document summarizes the production fixes applied for Vercel serverless deployment.

## What was failing

1. `ModuleNotFoundError: No module named 'flask_httpauth'`
2. `OSError: [Errno 30] Read-only file system: '/var/task/logs/microblog.log'`
3. Account creation could fail when SQLite attempted to write inside the project path (`/var/task` on Vercel is read-only).

## Changes made

### 1) Runtime pinned for Vercel
File: `vercel.json`

- Updated to `version: 3`
- Added function runtime pin:
  - `api/wsgi.py` -> `python3.12`

This avoids runtime drift and improves package compatibility.

### 2) Requirements cleanup
File: `requirements.txt`

- Kept: `Flask-HTTPAuth==4.7.0`
- Removed duplicate unpinned line: `flask_httpauth`

This ensures deterministic dependency install.

### 3) Database path fix for serverless
File: `config.py`

`SQLALCHEMY_DATABASE_URI` now resolves in this order:

1. `DATABASE_URL` (preferred for production)
2. If running on Vercel without `DATABASE_URL`: fallback to `sqlite:////tmp/app.db`
3. Local dev fallback: `sqlite:///.../app.db`

`/tmp` is writable on Vercel; project directory is not.

### 4) Logging safety for serverless
File: `app/__init__.py`

- Uses stdout logging when on Vercel or `LOG_TO_STDOUT` is enabled.
- Falls back to stdout if file logging fails.

## Recommended production setup

Set this in Vercel Project Settings -> Environment Variables:

- `SECRET_KEY` = strong random string
- `DATABASE_URL` = managed Postgres URL (recommended)
- Optional: `LOG_TO_STDOUT=1`

> Note: `/tmp` SQLite works as a fallback but is ephemeral. For real accounts/data, use Postgres via `DATABASE_URL`.

## Deploy steps

```powershell
git add .
git commit -m "Fix Vercel runtime, logging, and DB fallback"
git push
```

Then redeploy and check logs in Vercel.
