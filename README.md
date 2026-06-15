# Shared Expenses App

A Splitwise-style app for a flat-share group, built to handle a real,
messy expense spreadsheet (`expenses_export.csv`) with 19 distinct data
problems.

## Stack

- **Backend**: Django 6 + Django REST Framework, JWT auth (SimpleJWT)
- **Database**: SQLite (local dev) / PostgreSQL (Render, production)
- **Frontend**: React (Vite)
- **Deployment**: Backend on Render, Frontend on Vercel

## AI tool used

Claude (Anthropic) was used as the primary development collaborator for
scaffolding, the import pipeline, and balance logic. See `AI_USAGE.md`
for prompts and corrections.

## Project structure

```
backend/
  config/          - Django settings, root URLs
  accounts/        - custom User model (supports guest participants)
  groups/          - Group, GroupMembership (membership has join/leave dates)
  expenses/        - Expense, ExpenseSplit, Payment + balance/split logic
  imports/         - CSV import pipeline, anomaly detection, review queue
frontend/          - React app (Vite)
```

## Setup — Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt

# create .env (see .env.example) — at minimum:
#   SECRET_KEY=...
#   DEBUG=True
#   (leave DATABASE_URL unset to use local SQLite)

python manage.py migrate
python manage.py seed_group        # creates Aisha, Rohan, Priya, Meera, Sam
                                    # + the "Flatmates" group with membership history
python manage.py createsuperuser   # optional, for /admin

python manage.py runserver
```

API root: `http://127.0.0.1:8000/api/`
Admin: `http://127.0.0.1:8000/admin/`

### Importing the CSV

Once the server is running and you're authenticated as a group member,
`POST` the CSV file to:

```
POST /api/groups/{group_id}/import/
```

This creates an `ImportBatch`, auto-imports clean rows as `Expense` +
`ExpenseSplit` records, and creates a `Review Queue` entry (`ImportRow`,
status=`pending`) for every row that needs a human decision. The full
list of detected issues per row is the **Import Report**
(`GET /api/import-batches/{id}/report/`).

## Setup — Frontend

```bash
cd frontend
npm install
npm run dev
```

Set `VITE_API_BASE_URL` in `.env` to point at the backend
(`http://127.0.0.1:8000/api` for local dev).

## Key documents

- `SCOPE.md` — every anomaly found in `expenses_export.csv`, how it's
  handled, and the datbase schema
- `DECISIONS.md` — decision log: options considered and why
- `AI_USAGE.md` — AI prompts used and where AI output was wrong/corrected

## Deployment

- **Backend (Render)**: Web Service from `backend/`, build command
  `pip install -r requirements.txt && python manage.py migrate`, start
  command `gunicorn config.wsgi`. Attach a Render Postgres instance and
  set `DATABASE_URL` (auto-provided by Render when linked).
- **Frontend (Vercel)**: import `frontend/`, set `VITE_API_BASE_URL` to
  the deployed Render backend URL.

Live URLs: _TBD — added once deployed._
