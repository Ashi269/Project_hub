# Collaborative Project Management Platform

React (Vite) + FastAPI + MySQL monorepo implementing the PRD: role-based
authorization on **every** endpoint, membership-scoped project access, and
optimistic locking on tasks (HTTP 409 with the latest state).

```
backend/     FastAPI app (SQLAlchemy 2.0, Alembic, pytest)
frontend/    React 18 + Vite + React Router + Axios + TanStack Query
database/    Plain SQL schema + seed (schema.sql is the source of truth)
```

## 1. Quick start

### Prerequisites
- Python 3.11+ and Node 18+
- MySQL 8 (or MariaDB 10.4+) listening on `localhost:3306`

### Database

```bash
mysql -u root -p < database/schema.sql   # creates project_mgmt + tables
```

### Backend

```bash
cd backend
python -m venv venv && source venv/Scripts/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                  # then edit the values
python -m app.scripts.seed                            # demo users + sample project
python -m uvicorn app.main:app --reload --port 8000
```

API docs: http://localhost:8000/docs · Health: http://localhost:8000/health

Migrations mirror `database/schema.sql`:

```bash
cd backend
python -m alembic upgrade head     # apply
python -m alembic stamp head       # mark an existing DB as migrated
```

### Frontend

```bash
cd frontend
npm install
npm run dev        # http://localhost:5173 (proxies /api to :8000)
```

## 2. Environment variables

| Backend (`backend/.env`) | Default | Purpose |
|---|---|---|
| `DATABASE_URL` | `mysql+pymysql://root:@localhost:3306/project_mgmt?charset=utf8mb4` | SQLAlchemy connection string |
| `JWT_SECRET` | — | HS256 signing key; use a long random string |
| `JWT_EXPIRE_MINUTES` | `60` | Access-token lifetime |
| `CORS_ORIGINS` | `http://localhost:5173` | Comma-separated allow-list (dev proxy makes this optional) |

| Frontend (`frontend/.env`) | Default | Purpose |
|---|---|---|
| `VITE_API_URL` | `/api/v1` | Base URL; the Vite dev proxy forwards `/api` to `127.0.0.1:8000` |

## 3. Accounts

Two ways in:

1. **Sign up** at `/signup` — creates the account with the role you pick and
   signs you in immediately. (`POST /auth/register`, no admin needed.)
2. **Seeded demo accounts** — all share the password **`Password123!`**.

| Email | Global role |
|---|---|
| `admin@example.com` | Admin (sees everything, manages users, archives projects) |
| `manager@example.com` | Project Manager (manages team + tasks in their project) |
| `member@example.com` | Team Member (edits own assigned tasks, comments) |
| `member2@example.com` | Team Member |
| `viewer@example.com` | Viewer (read-only) |

They are members of the sample project **“Launch Website”** (5 tasks, 3 comments).

## 4. Architecture

```
page (React) -> api/*.js (Axios + JWT) -> router -> deps (identity/permission)
             -> service (business rules + version check) -> model -> MySQL
```

- **`backend/app/api/deps.py`** — the gate order is: 401 identity → 404
  existence/membership (non-members never learn a project exists) → 403
  permission → 409 archived-project mutation block.
- **`backend/app/permissions/policy.py`** — the permission matrix as pure
  functions; the only authority. The frontend mirror in
  `frontend/src/utils/permissions.js` is a UI hint only.
- **`backend/app/services/task_service.py`** — the single place optimistic
  locking lives: `UPDATE tasks SET … , version = version + 1 WHERE id = ?
  AND version = ?`; zero rows affected → **409 VERSION_CONFLICT** carrying the
  current task in `error.details.current`. Deletes check the version too.
- **`frontend/src/components/tasks/ConflictDialog.jsx`** — the 409 UI: shows
  latest server values next to your edits with *Reload latest* / *Reapply my
  changes*.
- Error format is uniform: `{"error": {"code", "message", "details"}}` for
  401 / 403 / 404 / 409 / 422.

### Key API routes

| Area | Routes |
|---|---|
| Auth | `POST /auth/login`, `POST /auth/register`, `GET/PATCH /auth/me`, `POST /auth/change-password` |
| Users (Admin) | `GET/POST /users`, `PATCH /users/{id}`, `POST /users/{id}/deactivate\|activate` |
| Projects | `GET/POST /projects`, `GET/PATCH /projects/{id}`, `POST /projects/{id}/archive` |
| Team | `GET/POST /projects/{id}/members`, `PATCH/DELETE …/members/{user_id}` |
| Tasks | `GET/POST /projects/{id}/tasks`, `GET/PATCH/DELETE /tasks/{id}`, `POST /tasks/{id}/status\|assign` |
| Comments | `GET/POST /tasks/{id}/comments`, `DELETE /comments/{id}` |
| Dashboard | `GET /dashboard` |

## 5. Roles and authority

Each role gets different authority, enforced on the server on every request
(and mirrored in the UI only to hide/disable buttons):

| Capability | Admin | Project Manager | Team Member | Viewer |
|---|---|---|---|---|
| Browse projects/tasks/comments they belong to | ✔ (all projects) | ✔ | ✔ | ✔ |
| Create projects | ✔ | ✘ | ✘ | ✘ |
| Edit project details | ✔ | ✔ (own projects) | ✘ | ✘ |
| Archive project | ✔ | ✘ | ✘ | ✘ |
| Manage team (add/remove/change role) | ✔ | ✔ (own projects) | ✘ | ✘ |
| Create / assign / delete tasks | ✔ | ✔ (own projects) | ✘ | ✘ |
| Edit task, change status | ✔ | ✔ (any task) | ✔ (own assigned only) | ✘ |
| Comment | ✔ | ✔ | ✔ | ✘ |
| Delete comments | ✔ | ✔ | own only | ✘ |
| Manage users (create, change role, deactivate) | ✔ | ✘ | ✘ | ✘ |

Signup allows choosing any of the four roles (demo convenience); in a
production deployment you would fix `role` to `team_member` in
`RegisterRequest` and let an admin promote accounts.

## 6. Tests

```bash
cd backend
python -m pytest
```

Runs against a dedicated `project_mgmt_test` database (created and rebuilt
automatically; your dev data is untouched). 32 tests cover:

- auth: login, wrong password, inactive accounts (token rejected), password change
- signup: role choice honored, duplicate email → 409, validation errors
- permissions: at least one negative test per role per endpoint group,
  non-member → 404 (no existence leak), archived project → 409
- concurrency: two clients with the same task `version` → one 200, one 409
  with the latest task in the body; stale delete also conflicts
- dashboard analytics: totals, priority mix, completion rate, per-assignee
  load, membership scoping

## 7. AI-usage disclosure

Generated with AI assistance (Codebuff):

- **AI-generated:** project scaffolding (backend package layout, frontend
  Vite/React structure), SQLAlchemy models + Alembic migration, all endpoint,
  service and permission code, the React page suite, CSS, the pytest suite and
  this README.
- **Human decisions:** stack selection (React/Vite, FastAPI, MySQL), the
  permission matrix itself (who may do what), the 404-vs-403 policy (no
  existence leak), optimistic-locking contract (`version` in every task
  mutation, 409 body shape), and the phased build order in `TODO.md`.
