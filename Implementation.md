# ClubTracker implementation guide

This guide translates **`CSHS Project Outline.pdf`** (4 pages: tasks B1–B4 and F1–F5) into a concrete, file-by-file plan for the starter repository at `https://github.com/P2PCSHS/ClubTracker.git`, branch `arena/0a7dd519-clubtracker`, base commit `533dbbb`.

**Read this first**

- For **Python files**, the outline asked for **detailed pseudocode**. Those blocks are intentionally **not paste-ready Python**; they describe every import, object, field, query, branch, and return value you need to write. Each Python section has a **Learn** list of official documentation links.
- For **non-Python files** (`requirements.txt`, YAML, JSON, HTML, CSS, TypeScript/TSX, shell commands), the blocks are **paste-ready code** with an explicit “where” instruction.
- The TypeScript/TSX snippets below are one complete, mutually consistent frontend. They were checked in a scratch copy with `npm run lint`, `npm run typecheck`, `npm run format:check`, and `npm run build` against `react-router-dom@7.18.5`.
- This document is a **guide/specification**, not an applied implementation. It does not change the app source files.

## 1. What the starter repo already gives you

| Existing file | What it does now | What the outline changes |
| --- | --- | --- |
| `backend/app/__init__.py` | `create_app()` loads `Config`, enables CORS, registers the `api` blueprint at `/api` | Add SQLAlchemy + JWT wiring, register the auth blueprint, add `init-db` CLI command |
| `backend/app/config.py` | Reads `FRONTEND_ORIGIN` | Add database URI, JWT secret/expiry, and a `TestConfig` |
| `backend/app/routes.py` | `/api/hello` and `/api/health` | Delete `/hello`; add club/member/event/attendance/hours routes; keep `/health` public |
| `backend/tests/test_routes.py` | Tests `/api/hello` and `/api/health` | Remove the hello test; add authenticated API tests |
| `frontend/src/api.ts` | `apiGet()` and `getHello()` | Replace with a typed `apiFetch()` that attaches `Authorization: Bearer <token>` |
| `frontend/src/App.tsx` | Boilerplate “Hello from React!” | Replace with `<Routes>`, login, protected routes, dashboard, club view, attendance tracker |
| `frontend/src/main.tsx` | Mounts `<App />` | Wrap in `<BrowserRouter>` and `<AuthProvider>` |
| `frontend/vite.config.ts` | Proxies `/api` to Flask | **No change required** |
| `frontend/nginx.conf` | Proxies `/api` to the backend container | **No change required** |
| `docker-compose.yml` | `backend` + `frontend` only | Add Postgres `db`, volume, healthcheck, backend env/dependency |
| `.github/workflows/ci.yml` | Runs backend and frontend checks | Add test env vars so pytest can use SQLite without a Postgres service |

## 2. Conventions used by every snippet

- JSON keys are **snake_case** and match the database columns: `user_id`, `club_id`, `hours_worth`, `hours_awarded`, `is_leader`.
- `date` is an ISO date string: `YYYY-MM-DD`.
- `hours_worth` / `hours_awarded` / `total_hours` are JSON numbers.
- `User` API objects **never** include `password_hash`.
- All data endpoints require a valid JWT. `/api/health` and `POST /api/auth/login` are public; Flask-CORS handles `OPTIONS` preflights.
- `POST /api/clubs`, `POST /api/clubs/<id>/members`, `POST /api/clubs/<id>/events`, and `POST /api/events/<id>/attendance` require a leader **and** the leader of that specific club/event.
- Lists return arrays directly (`Club[]`, `User[]`, `ClubEvent[]`).
- The Python class is named `Event` to match the outline; the TypeScript interface is named `ClubEvent` so it does not collide with the DOM `Event` type.
- JWT local storage key: `clubtracker_token`. The outline requires `localStorage`; for a real production app, research HttpOnly cookies and XSS risks.

## 3. Shared API contract

The frontend code in F1–F5 assumes exactly these responses. Supporting endpoints are marked “gap” because the outline’s UI needs them but does not list them.

| Method | Path | Auth | Request body | Success response |
| --- | --- | --- | --- | --- |
| `GET` | `/api/health` | public | — | `{ "status": "ok" }` |
| `POST` | `/api/auth/login` | public | `{ email, password }` | `{ access_token, user: User }` |
| `GET` | `/api/auth/me` | JWT | — | `User` *(gap: lets `AuthContext` restore a session)* |
| `GET` | `/api/clubs` | JWT | — | `Club[]` |
| `POST` | `/api/clubs` | leader | `{ name, description }` | `Club`, `201` |
| `GET` | `/api/me/clubs` | JWT | — | `Club[]` *(gap: Dashboard needs “my clubs”)* |
| `GET` | `/api/clubs/<club_id>` | JWT | — | `Club` *(gap: ClubView needs details)* |
| `GET` | `/api/clubs/<club_id>/members` | JWT | — | `User[]` |
| `POST` | `/api/clubs/<club_id>/members` | club leader | `{ user_id }` | `ClubMember`, `201` |
| `GET` | `/api/users` | JWT | — | `User[]` *(gap: AddMemberForm dropdown)* |
| `GET` | `/api/clubs/<club_id>/events` | JWT | — | `ClubEvent[]` |
| `POST` | `/api/clubs/<club_id>/events` | club leader | `{ name, date, hours_worth }` | `ClubEvent`, `201` |
| `GET` | `/api/events/<event_id>` | JWT | — | `ClubEvent` *(gap: AttendanceTracker needs `club_id`)* |
| `GET` | `/api/events/<event_id>/attendance` | JWT | — | `AttendanceRecord[]` *(gap: prefill checkboxes)* |
| `POST` | `/api/events/<event_id>/attendance` | club leader | `{ attendance: [{ user_id, hours_awarded }] }` | `{ saved: number }` |
| `GET` | `/api/users/<user_id>/hours` | self or leader | — | `HoursSummary` |

Shapes (also declared in `frontend/src/api.ts`):

```text
User              = { id, name, email, is_leader }
Club              = { id, name, description|null, leader_id|null }
ClubEvent         = { id, club_id, name, date, hours_worth }
ClubMember        = { user_id, club_id }  # mapping table has a composite PK
AttendanceRecord  = { id, event_id, user_id, hours_awarded }
HoursEntry        = { event_id, event_name, date, hours_awarded }
HoursSummary      = { user_id, total_hours, entries: HoursEntry[] }
```

`POST /api/events/<event_id>/attendance` is a **full replacement** for that event. Send one entry per club member: `hours_awarded > 0` means present; `0` means absent. The backend deletes records for omitted or zero-hour users so unchecking a box actually removes attendance.

## 4. Outline gaps this guide fills

| Gap in outline | Why it matters | Resolution |
| --- | --- | --- |
| No `JWT_SECRET_KEY`, `DATABASE_URL`, or Postgres env variables | JWT cannot be signed; backend cannot reach Postgres | `.env.example`, `config.py`, `docker-compose.yml` |
| No `backend/app/extensions.py` | Defining `db` inside `__init__.py` and importing it from `models.py` creates circular imports | Add a tiny extensions module |
| No schema creation or first users | There is no registration endpoint, so nobody can log in | `flask --app wsgi init-db` CLI command seeds one leader and one member |
| Only POST routes are marked protected | B2 goal says “only logged-in users can access data” | Add `@jwt_required()` to every data `GET` |
| `@leader_required` checks only a global flag | A leader of Club A should not edit Club B | Decorator also checks `club_id` / `event_id` ownership |
| Dashboard needs “clubs the user is part of (or leads)” | `GET /api/clubs` returns **all** clubs | Add `GET /api/me/clubs` |
| AddMemberForm needs a student dropdown | No user-list endpoint is listed | Add `GET /api/users` |
| AttendanceTracker needs event details and existing checkboxes | Only a POST attendance endpoint is listed | Add `GET /api/events/<id>` and `GET /api/events/<id>/attendance` |
| `GET /api/hello` is deleted by B3 | Existing test and CI would fail | Update `test_routes.py`, add `conftest.py`, set CI env |
| No migrations | `db.create_all()` cannot alter existing tables | Use `init-db` for this learning build; use Flask-Migrate/Alembic for schema changes later |

## 5. Recommended execution order

1. Apply B1 dependency/config/Compose changes.
2. Write B1 Python pseudocode as real Python (`extensions.py`, `config.py`, `__init__.py`, `models.py`).
3. Run the one-time `init-db` command and confirm two demo users exist.
4. Apply B2 (`auth.py`, `utils.py`, already-mentioned config/wiring) and test login with `curl` or HTTPie.
5. Apply B3 and B4 (`routes.py`), then update tests/CI.
6. Apply F1 → F2 → F3 → F4 → F5; create `frontend/src/context`, `frontend/src/pages`, and `frontend/src/components`.
7. Run `make format`, `make lint`, `make test`, then `make dev`.

---

# Backend tasks

## Task B1 — Database integration & models

### B1.1 `backend/requirements.txt` — direct code

**Where:** replace the whole file. `Flask-JWT-Extended` is included here so B1 and B2 need only one dependency edit. `werkzeug.security` needs **no** extra line because Flask already depends on Werkzeug.

```text
flask>=3.0
flask-cors>=5.0
flask-jwt-extended>=4.7
flask-sqlalchemy>=3.1
gunicorn>=23.0
psycopg2-binary>=2.9
```

**Also:** `backend/requirements-dev.txt` stays unchanged because it already starts with `-r requirements.txt`.

### B1.2 `docker-compose.yml` — direct code

**Where:** replace the whole file. This adds `postgres:15`, a persistent named volume, a healthcheck, the backend `DATABASE_URL`, `JWT_SECRET_KEY`, and `depends_on: db: condition: service_healthy`. The host port is bound to `127.0.0.1` so Postgres is not exposed publicly; remove the binding or change `POSTGRES_PORT` if port 5432 is already in use. Use a URL-safe `POSTGRES_PASSWORD`: `@`, `:`, `/`, and `#` must be URL-encoded inside `DATABASE_URL`. The `$${POSTGRES_USER}` / `$${POSTGRES_DB}` in the healthcheck is intentional; Compose passes a literal `${POSTGRES_USER}` / `${POSTGRES_DB}` to the container shell.

```yaml
services:
  db:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: "${POSTGRES_DB:-clubtracker}"
      POSTGRES_USER: "${POSTGRES_USER:-clubtracker}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-clubtracker_dev_password}"
    ports:
      - "127.0.0.1:${POSTGRES_PORT:-5432}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  backend:
    build: ./backend
    restart: unless-stopped
    environment:
      FRONTEND_ORIGIN: "${FRONTEND_ORIGIN:-*}"
      DATABASE_URL: "postgresql+psycopg2://${POSTGRES_USER:-clubtracker}:${POSTGRES_PASSWORD:-clubtracker_dev_password}@db:5432/${POSTGRES_DB:-clubtracker}"
      JWT_SECRET_KEY: "${JWT_SECRET_KEY:-change-me-in-production}"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:5000/api/health')"]
      interval: 30s
      timeout: 5s
      retries: 3

  frontend:
    build: ./frontend
    restart: unless-stopped
    ports:
      - "127.0.0.1:${APP_PORT:-8080}:80"
    depends_on:
      - backend

volumes:
  postgres_data:
```

**Learn:** [Docker Compose startup order / healthchecks](https://docs.docker.com/compose/how-tos/startup-order/), [official Postgres image environment variables](https://hub.docker.com/_/postgres).

### B1.3 `.env.example` — direct code, supporting change

**Where:** replace the whole file. Then run `cp .env.example .env` and replace both `replace-with-...` values before any real deployment. `.env` is already git-ignored.

```dotenv
# Copy this file to .env and edit as needed. .env is git-ignored.

# Port the whole app is exposed on when running with docker compose.
APP_PORT=8080

# Origin allowed to call the API. Leave as * for local development.
# In production behind nginx the frontend and API share an origin, so this
# only matters if you host them separately.
FRONTEND_ORIGIN=*

# PostgreSQL settings used by the db service and the backend container.
POSTGRES_DB=clubtracker
POSTGRES_USER=clubtracker
POSTGRES_PASSWORD=replace-with-a-long-random-local-password
POSTGRES_PORT=5432

# Secret used to sign JWT access tokens. Generate a new random value for
# every real deployment; never commit the production value.
JWT_SECRET_KEY=replace-with-a-long-random-secret

# Optional: set this when running Flask outside Docker to override the
# default SQLite URL. docker-compose.yml constructs the PostgreSQL URL for you.
# DATABASE_URL=postgresql+psycopg2://clubtracker:password@localhost:5432/clubtracker
```

### B1.4 `.gitignore` — direct code, supporting change

**Where:** add `instance/` and `*.db` under the Python section. The full resulting file is shown so SQLite test databases are never committed.

```gitignore
# Python
__pycache__/
*.pyc
.venv/
.pytest_cache/
.ruff_cache/
instance/
*.db

# Node
node_modules/
frontend/dist/

# Environment
.env

# Editors / OS
.idea/
.DS_Store
```

### B1.5 `backend/app/config.py` — detailed pseudocode

**Where:** keep the existing module docstring and `import os`; add a `timedelta` import concept; replace the `Config` class body and append `TestConfig`.

Important: Flask-SQLAlchemy reads `SQLALCHEMY_DATABASE_URI` when `db.init_app(app)` runs, so all values must be class attributes before `create_app()` loads the class. The SQLite default is only for local development and pytest; Docker supplies PostgreSQL through `DATABASE_URL`.

```text
IMPORT timedelta FROM datetime
IMPORT os  # already present

CLASS Config:
    FRONTEND_ORIGIN = READ ENV "FRONTEND_ORIGIN", DEFAULT "*"

    # Required by Flask-SQLAlchemy.
    SQLALCHEMY_DATABASE_URI = READ ENV "DATABASE_URL",
                              DEFAULT "sqlite:///clubtracker.db"
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    # Required by Flask-JWT-Extended.
    JWT_SECRET_KEY = READ ENV "JWT_SECRET_KEY",
                     DEFAULT "dev-only-change-me"
    SECRET_KEY = JWT_SECRET_KEY
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(hours=8)

CLASS TestConfig INHERITS Config:
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///test.db"
    JWT_SECRET_KEY = "test-only-secret"
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(hours=1)
```

**Learn:** [Flask configuration handling](https://flask.palletsprojects.com/en/stable/config/), [Flask-SQLAlchemy configuration keys](https://flask-sqlalchemy.readthedocs.io/en/stable/config/), [Flask-JWT-Extended configuration options](https://flask-jwt-extended.readthedocs.io/en/stable/options.html), [`os.environ`](https://docs.python.org/3/library/os.html#os.environ), [`datetime.timedelta`](https://docs.python.org/3/library/datetime.html#timedelta-objects).

### B1.6 `backend/app/extensions.py` — NEW FILE, detailed pseudocode

**Where:** create this new module. It exists only to hold extension objects so `models.py`, `auth.py`, `routes.py`, and `utils.py` can import them without circular imports.

```text
IMPORT SQLAlchemy FROM flask_sqlalchemy
IMPORT JWTManager FROM flask_jwt_extended

CREATE module-level object db = SQLAlchemy()
CREATE module-level object jwt = JWTManager()

# Do NOT call db.init_app() or jwt.init_app() here.
# create_app() will do that after app.config is loaded.
```

**Learn:** [Flask application factories](https://flask.palletsprojects.com/en/stable/patterns/appfactories/), [Flask extension development / `init_app` pattern](https://flask.palletsprojects.com/en/stable/extensiondev/), [Flask-SQLAlchemy Quick Start](https://flask-sqlalchemy.readthedocs.io/en/stable/quickstart/).

### B1.7 `backend/app/__init__.py` — detailed pseudocode

**Where:** replace the whole file. This one factory satisfies B1’s “initialize `db` and attach it” **and** B2’s “initialize `JWTManager(app)`” and “register the auth blueprint”. B2 therefore has no second `__init__.py` edit.

```text
IMPORT Flask FROM flask
IMPORT CORS FROM flask_cors
IMPORT Config FROM .config
IMPORT db, jwt FROM .extensions
IMPORT api FROM .routes
IMPORT auth_bp FROM .auth
IMPORT generate_password_hash FROM werkzeug.security
IMPORT models module FROM .models  # or IMPORT User, Club, club_members FROM .models

FUNCTION create_app(config_class = Config) RETURNS Flask:
    app = Flask(__name__)
    LOAD app.config FROM config_class

    CORS(app, RESOURCES = {"/api/*": {"origins": app.config["FRONTEND_ORIGIN"]}})

    CALL db.init_app(app)
    CALL jwt.init_app(app)

    # Importing .models above registers every table with SQLAlchemy metadata.
    REGISTER api blueprint WITH url_prefix "/api"
    REGISTER auth_bp blueprint WITH url_prefix "/api/auth"

    DEFINE CLI COMMAND "init-db":
        # Flask CLI commands automatically run inside an application context.
        CALL db.create_all()

        IF SELECT COUNT(User) == 0:
            leader = User(
                name = "Demo Leader",
                email = "leader@example.com",
                password_hash = generate_password_hash("leader123"),
                is_leader = True
            )
            member = User(
                name = "Demo Member",
                email = "member@example.com",
                password_hash = generate_password_hash("member123"),
                is_leader = False
            )
            ADD leader AND member TO db.session
            FLUSH db.session              # assigns leader.id without ending the transaction

            club = Club(
                name = "Robotics",
                description = "Builds robots",
                leader_id = leader.id
            )
            APPEND leader AND member TO club.members
            ADD club TO db.session
            COMMIT db.session
            PRINT "Created demo leader, member, and Robotics club"

        PRINT "Database ready"

    RETURN app
```

Notes to translate into Python:

- Import `.models` inside `create_app()` (or at module level **after** `db` is importable) so `db.create_all()` sees all tables.
- `generate_password_hash()` is from `werkzeug.security`; never store plaintext passwords.
- Demo credentials are `leader@example.com / leader123` and `member@example.com / member123`; `init-db` also seeds a Robotics club with both users as members. Change or remove them before deployment.
- `db.create_all()` creates missing tables only. For later column changes, add `Flask-Migrate` and `flask db migrate` / `flask db upgrade`.

**Learn:** [Flask application factories](https://flask.palletsprojects.com/en/stable/patterns/appfactories/), [Flask CLI custom commands](https://flask.palletsprojects.com/en/stable/cli/), [Flask-SQLAlchemy `create_all()` and app context](https://flask-sqlalchemy.readthedocs.io/en/stable/quickstart/), [Werkzeug password hashing](https://werkzeug.palletsprojects.com/en/stable/utils/), [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/).

### B1.8 `backend/app/models.py` — NEW FILE, detailed pseudocode

**Where:** create this file. The outline calls the mapping table `ClubMember`; the simplest SQLAlchemy implementation is a `db.Table` named `club_members` with a composite primary key. If your instructor requires a class, wrap the same two columns in `class ClubMember(db.Model)` with an extra `id` primary key and a `UniqueConstraint(user_id, club_id)`.

```text
IMPORT db FROM .extensions
# Conceptually use these SQLAlchemy names: Integer, String, Text, Float/Date,
# ForeignKey, UniqueConstraint, Table, Column, relationship.

MAPPING TABLE club_members  # this is the outline's ClubMember
    COLUMN user_id  -> ForeignKey("users.id"), primary_key=True
    COLUMN club_id  -> ForeignKey("clubs.id"), primary_key=True

MODEL User, TABLE "users":
    id             = Integer, primary key
    name           = String(120), not null
    email          = String(255), unique, not null, indexed
    password_hash  = String(255), not null
    is_leader      = Boolean, not null, default False

    led_clubs      = relationship to Club where Club.leader_id == User.id
    clubs          = many-to-many relationship to Club through club_members,
                     back_populates = "members"
    attendance     = relationship to Attendance, cascade delete-orphan

    METHOD to_public_dict():
        RETURN { id, name, email, is_leader }
        # NEVER include password_hash.

MODEL Club, TABLE "clubs":
    id            = Integer, primary key
    name          = String(150), unique, not null
    description   = Text, nullable
    leader_id     = Integer, ForeignKey("users.id"), not null

    leader        = relationship to User (the club leader)
    members       = many-to-many relationship to User through club_members,
                    back_populates = "clubs"
    events        = relationship to Event, cascade delete-orphan

    METHOD to_dict():
        RETURN { id, name, description, leader_id }

MODEL Event, TABLE "events":
    id            = Integer, primary key
    club_id       = Integer, ForeignKey("clubs.id"), not null
    name          = String(150), not null
    date          = Date, not null
    hours_worth   = Float, not null, default 0

    club          = relationship to Club
    attendance    = relationship to Attendance, cascade delete-orphan

    METHOD to_dict():
        RETURN {
            id,
            club_id,
            name,
            date = ISO string "YYYY-MM-DD",
            hours_worth = float(hours_worth)
        }

MODEL Attendance, TABLE "attendance":
    id             = Integer, primary key
    event_id       = Integer, ForeignKey("events.id"), not null
    user_id        = Integer, ForeignKey("users.id"), not null
    hours_awarded  = Float, not null, default 0

    TABLE-LEVEL UniqueConstraint(event_id, user_id, name="uq_attendance_event_user")

    event          = relationship to Event
    user           = relationship to User

    METHOD to_dict():
        RETURN { id, event_id, user_id, hours_awarded = float(hours_awarded) }
```

Design choices to preserve:

- `UNIQUE(event_id, user_id)` makes the B4 “create or update” logic deterministic.
- `Club.leader_id` is a foreign key to `User`; `is_leader` is a global permission flag.
- The creator of a club should also be appended to `club.members` so the Dashboard shows it.
- Use `Float` for a class project. If you need exact decimal accounting, use `Numeric(5, 2)` and convert `Decimal` to `float` inside every `to_dict()`.
- Cascade from `Club → Event → Attendance` keeps orphan rows from accumulating.

**Learn:** [Flask-SQLAlchemy models and tables](https://flask-sqlalchemy.readthedocs.io/en/stable/models/), [SQLAlchemy ORM relationships](https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html), [SQLAlchemy many-to-many](https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html#many-to-many), [SQLAlchemy unique constraints](https://docs.sqlalchemy.org/en/20/core/constraints.html#unique-constraint), [Python `datetime.date.fromisoformat`](https://docs.python.org/3/library/datetime.html#datetime.date.fromisoformat).

### B1.9 Commands for B1

```bash
cd backend
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
.venv/bin/flask --app wsgi init-db
```

For Docker instead:

```bash
cp .env.example .env
# edit JWT_SECRET_KEY and POSTGRES_PASSWORD
docker compose up --build -d
docker compose exec backend flask --app wsgi init-db
```

---

## Task B2 — Authentication & authorization

### B2.1 `backend/requirements.txt` — direct code

**Where:** no second edit. `flask-jwt-extended>=4.7` was already added in B1.1. The outline allows `bcrypt` **or** `werkzeug.security`; this guide uses `werkzeug.security`, which is already installed as a Flask dependency. If you choose `bcrypt` instead, add `bcrypt>=4.0` and replace the two `generate_password_hash` / `check_password_hash` concepts in the pseudocode.

### B2.2 `backend/app/config.py` — pseudocode cross-reference

**Where:** no second edit. `JWT_SECRET_KEY`, `SECRET_KEY`, and `JWT_ACCESS_TOKEN_EXPIRES` were added in B1.5. Never ship the default `dev-only-change-me` secret.

### B2.3 `backend/app/__init__.py` — pseudocode cross-reference

**Where:** no second edit. `jwt.init_app(app)` and `app.register_blueprint(auth_bp, url_prefix="/api/auth")` are in the B1.7 factory.

### B2.4 `backend/app/auth.py` — NEW FILE, detailed pseudocode

**Where:** create this file and register its blueprint as shown in B1.7. The blueprint’s own routes are `/login` and `/me`; the factory prefix makes them `/api/auth/login` and `/api/auth/me`.

```text
IMPORT Blueprint, jsonify, request FROM flask
IMPORT create_access_token, get_jwt_identity, jwt_required FROM flask_jwt_extended
IMPORT func FROM sqlalchemy  # for case-insensitive email lookup
IMPORT check_password_hash FROM werkzeug.security
IMPORT db FROM .extensions
IMPORT User FROM .models
IMPORT json_error FROM .utils

CREATE auth_bp = Blueprint("auth", __name__)

HELPER FUNCTION public_user(user):
    RETURN user.to_public_dict()

ROUTE POST "/login" (public):
    body = request.get_json(silent=True) OR empty dictionary
    email = TRIM(body["email"]) as string, DEFAULT ""
    password = body["password"] as string, DEFAULT ""

    IF email is empty OR password is empty:
        RETURN json_error("Email and password are required", 400)

    user = QUERY User WHERE lower(User.email) == lower(email), FIRST RESULT

    IF user does not exist OR check_password_hash(user.password_hash, password) is False:
        RETURN json_error("Invalid email or password", 401)
        # Use the same message for both cases so you do not reveal which emails exist.

    claims = {
        "is_leader": user.is_leader,
        "email": user.email
    }
    access_token = create_access_token(
        identity = STRING(user.id),        # JWT "sub" should be a string
        additional_claims = claims
    )

    RETURN 200 JSON {
        "access_token": access_token,
        "user": public_user(user)
    }

ROUTE GET "/me" (REQUIRE valid JWT):
    current_user_id = INTEGER(get_jwt_identity())
    user = db.session.get(User, current_user_id)

    IF user does not exist:
        RETURN json_error("User not found", 404)

    RETURN 200 JSON public_user(user)
```

**Learn:** [Flask-JWT-Extended Basic Usage](https://flask-jwt-extended.readthedocs.io/en/stable/basic_usage.html), [Storing additional data in JWTs](https://flask-jwt-extended.readthedocs.io/en/stable/add_custom_data_claims.html), [Werkzeug security helpers](https://werkzeug.palletsprojects.com/en/stable/utils/), [Flask request data](https://flask.palletsprojects.com/en/stable/quickstart/).

### B2.5 `backend/app/utils.py` — NEW FILE, detailed pseudocode

**Where:** create this file. `leader_required` must call `verify_jwt_in_request()` itself, so routes using it should **not** also use `@jwt_required()`. It checks the claim first, then checks database ownership when the URL contains `club_id` or `event_id`.

```text
IMPORT wraps FROM functools
IMPORT jsonify FROM flask
IMPORT get_jwt, get_jwt_identity, verify_jwt_in_request FROM flask_jwt_extended
IMPORT db FROM .extensions
IMPORT User, Club, Event FROM .models

HELPER FUNCTION json_error(message, status):
    RETURN jsonify({"error": message}), status

HELPER FUNCTION current_user_id():
    RETURN INTEGER(get_jwt_identity())

FUNCTION leader_required(view_function):
    DEFINE wrapper(*args, **kwargs) WITH wraps(view_function):
        CALL verify_jwt_in_request()          # raises 401 if token missing/invalid
        claims = get_jwt()
        claims_leader = claims.get("is_leader", False)

        # Re-check the database too; a token claim can be stale.
        user = db.session.get(User, current_user_id())
        IF user does not exist OR user.is_leader is not True OR claims_leader is not True:
            RETURN json_error("Leader access required", 403)

        IF kwargs CONTAINS "club_id":
            club = db.session.get(Club, kwargs["club_id"])
            IF club does not exist:
                RETURN json_error("Club not found", 404)
            IF club.leader_id != user.id:
                RETURN json_error("You do not lead this club", 403)

        IF kwargs CONTAINS "event_id":
            event = db.session.get(Event, kwargs["event_id"])
            IF event does not exist:
                RETURN json_error("Event not found", 404)
            club = db.session.get(Club, event.club_id)
            IF club does not exist OR club.leader_id != user.id:
                RETURN json_error("You do not lead this club", 403)

        RETURN view_function(*args, **kwargs)
    RETURN wrapper

HELPER FUNCTION require_self_or_leader(target_user_id):
    CALL verify_jwt_in_request()
    requester_id = current_user_id()
    claims = get_jwt()
    IF requester_id == target_user_id:
        RETURN requester_id
    IF claims.get("is_leader", False) is True:
        RETURN requester_id
    RETURN json_error("You can only view your own hours", 403)
```

**Learn:** [Flask-JWT-Extended custom decorators](https://flask-jwt-extended.readthedocs.io/en/stable/custom_decorators.html), [JWT API reference](https://flask-jwt-extended.readthedocs.io/en/stable/api.html), [`functools.wraps`](https://docs.python.org/3/library/functools.html#functools.wraps), [OWASP authorization cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).

### B2.6 Quick login check

```bash
curl -s -X POST http://localhost:5000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"leader@example.com","password":"leader123"}'
```

Copy `access_token` from the response, then:

```bash
curl -s http://localhost:5000/api/clubs \
  -H "Authorization: Bearer PASTE_TOKEN_HERE"
```

---

## Task B3 — Club & Member API endpoints

### B3.1 `backend/app/routes.py` — detailed pseudocode, part 1

**Where:** replace the whole file eventually. Start with the imports/helpers below, keep `/api/health`, **delete `/api/hello`**, then add B3 routes. B4.1 appends more routes to this same file.

Imports and shared helpers:

```text
IMPORT Blueprint, jsonify, request FROM flask
IMPORT jwt_required, get_jwt_identity, get_jwt FROM flask_jwt_extended
IMPORT func, select FROM sqlalchemy  # or db.select / db.func via Flask-SQLAlchemy
IMPORT db FROM .extensions
IMPORT User, Club, Event, Attendance, club_members FROM .models
IMPORT json_error, current_user_id, leader_required, require_self_or_leader FROM .utils
IMPORT date FROM datetime

CREATE api = Blueprint("api", __name__)

HELPER FUNCTION get_club_or_404(club_id):
    club = db.session.get(Club, club_id)
    IF club does not exist: RETURN json_error("Club not found", 404)
    RETURN club

HELPER FUNCTION get_event_or_404(event_id):
    event = db.session.get(Event, event_id)
    IF event does not exist: RETURN json_error("Event not found", 404)
    RETURN event

HELPER FUNCTION public_user(user): RETURN user.to_public_dict()

HELPER FUNCTION parse_hours(value):
    TRY: number = float(value)
    IF number < 0 OR number IS NOT finite: RAISE ValueError
    RETURN number
    EXCEPT: RETURN json_error("hours_worth must be a non-negative number", 400)

ROUTE GET "/health" (public):
    RETURN 200 JSON { "status": "ok" }
```

B3 routes:

```text
ROUTE GET "/clubs" (REQUIRE valid JWT):
    clubs = SCALAR RESULT OF SELECT Club ORDER BY Club.name
    RETURN 200 JSON [ club.to_dict() FOR EACH club ]

ROUTE POST "/clubs" (leader_required):
    body = request.get_json(silent=True) OR empty dictionary
    name = TRIM(body["name"]) as string, DEFAULT ""
    description = TRIM(body["description"]) as string, DEFAULT ""

    IF name is empty: RETURN json_error("Club name is required", 400)

    leader_id = current_user_id()
    club = Club(name=name, description=description, leader_id=leader_id)
    ADD club TO db.session

    # Make the creator a member so GET /api/me/clubs includes the new club.
    leader = db.session.get(User, leader_id)
    APPEND leader TO club.members

    TRY COMMIT:
        RETURN 201 JSON club.to_dict()
    CATCH duplicate club name / IntegrityError:
        ROLLBACK
        RETURN json_error("A club with that name already exists", 409)

ROUTE GET "/me/clubs" (REQUIRE valid JWT)        # supporting endpoint
    user = db.session.get(User, current_user_id())
    led_clubs = SCALARS(SELECT Club WHERE Club.leader_id == user.id)
    member_clubs = user.clubs
    MERGE led_clubs + member_clubs INTO a list keyed by club.id (no duplicates)
    SORT by club.name
    RETURN 200 JSON [ club.to_dict() FOR EACH club ]

ROUTE GET "/clubs/<int:club_id>" (REQUIRE valid JWT)   # supporting endpoint
    result = get_club_or_404(club_id)
    IF result IS an error response: RETURN result
    RETURN 200 JSON result.to_dict()

ROUTE GET "/clubs/<int:club_id>/members" (REQUIRE valid JWT):
    club = get_club_or_404(club_id)
    IF club IS an error response: RETURN club
    users = club.members SORTED BY name
    RETURN 200 JSON [ public_user(user) FOR EACH user ]

ROUTE POST "/clubs/<int:club_id>/members" (leader_required):
    # leader_required already verified that this user leads club_id.
    body = request.get_json(silent=True) OR empty dictionary
    user_id = body["user_id"]
    IF user_id IS NOT an integer: RETURN json_error("user_id is required", 400)

    club = get_club_or_404(club_id)
    user = db.session.get(User, user_id)
    IF user does not exist: RETURN json_error("User not found", 404)
    IF user ALREADY IN club.members:
        RETURN json_error("User is already a member", 409)

    APPEND user TO club.members
    COMMIT db.session
    RETURN 201 JSON { "user_id": user_id, "club_id": club_id }

ROUTE GET "/users" (REQUIRE valid JWT)            # supporting endpoint
    users = SCALARS(SELECT User ORDER BY User.name)
    RETURN 200 JSON [ public_user(user) FOR EACH user ]
```

The outline calls `ClubMember` a mapping table. In B1.8 it is implemented as `club_members` with a composite primary key, so this endpoint and the `ClubMember` TypeScript interface use `{ user_id, club_id }` rather than a single mapping `id`.

**Learn:** [Flask blueprints and views](https://flask.palletsprojects.com/en/stable/blueprints/), [Flask-SQLAlchemy `select` queries](https://flask-sqlalchemy.readthedocs.io/en/stable/quickstart/), [SQLAlchemy 2.0 SELECT guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html), [Flask `jsonify` / responses](https://flask.palletsprojects.com/en/stable/quickstart/).

---

## Task B4 — Events, Attendance & Hours API

### B4.1 `backend/app/routes.py` — detailed pseudocode, part 2

**Where:** append these routes to the same `routes.py` from B3.1. Reuse the helpers above.

```text
ROUTE GET "/clubs/<int:club_id>/events" (REQUIRE valid JWT):
    club = get_club_or_404(club_id)
    IF club IS an error response: RETURN club
    events = SCALARS(
        SELECT Event WHERE Event.club_id == club_id ORDER BY Event.date, Event.id
    )
    RETURN 200 JSON [ event.to_dict() FOR EACH event ]

ROUTE POST "/clubs/<int:club_id>/events" (leader_required):
    body = request.get_json(silent=True) OR empty dictionary
    name = TRIM(body["name"]) as string, DEFAULT ""
    date_text = body["date"] as string, DEFAULT ""
    hours_worth = parse_hours(body["hours_worth"])

    IF name is empty: RETURN json_error("Event name is required", 400)
    TRY parsed_date = date.fromisoformat(date_text)
    EXCEPT ValueError: RETURN json_error("date must be YYYY-MM-DD", 400)
    IF hours_worth IS an error response: RETURN hours_worth

    event = Event(
        club_id = club_id,
        name = name,
        date = parsed_date,
        hours_worth = hours_worth
    )
    ADD event TO db.session
    COMMIT
    RETURN 201 JSON event.to_dict()

ROUTE GET "/events/<int:event_id>" (REQUIRE valid JWT)    # supporting endpoint
    event = get_event_or_404(event_id)
    IF event IS an error response: RETURN event
    RETURN 200 JSON event.to_dict()

ROUTE GET "/events/<int:event_id>/attendance" (REQUIRE valid JWT)  # supporting endpoint
    event = get_event_or_404(event_id)
    IF event IS an error response: RETURN event
    records = SCALARS(SELECT Attendance WHERE Attendance.event_id == event_id)
    RETURN 200 JSON [ record.to_dict() FOR EACH record ]

ROUTE POST "/events/<int:event_id>/attendance" (leader_required):
    # leader_required already used event.club_id to verify club ownership.
    body = request.get_json(silent=True) OR empty dictionary
    items = body["attendance"]
    IF items IS NOT a list: RETURN json_error("attendance must be an array", 400)

    event = get_event_or_404(event_id)
    IF event IS an error response: RETURN event

    club_member_ids = SET OF user ids FROM club_members WHERE club_id == event.club_id
    validated = EMPTY LIST
    seen_user_ids = EMPTY SET

    FOR EACH item IN items:
        user_id = item["user_id"]
        hours = parse_hours(item["hours_awarded"])
        IF hours IS an error response: ROLLBACK AND RETURN hours
        IF user_id IS NOT an integer: RETURN json_error("user_id is required", 400)
        IF user_id IN seen_user_ids: RETURN json_error("Duplicate user_id", 400)
        IF user_id NOT IN club_member_ids:
            RETURN json_error("User is not a member of this club", 400)
        IF hours > event.hours_worth:
            RETURN json_error("hours_awarded cannot exceed hours_worth", 400)
        ADD { user_id, hours_awarded: hours } TO validated
        ADD user_id TO seen_user_ids

    existing = MAP user_id -> Attendance row
               FOR SELECT Attendance WHERE Attendance.event_id == event_id
    submitted_ids = SET OF user_id FROM validated

    TRY:
        FOR EACH entry IN validated:
            IF entry.hours_awarded > 0:
                IF entry.user_id IN existing:
                    SET existing[entry.user_id].hours_awarded = entry.hours_awarded
                ELSE:
                    ADD NEW Attendance(
                        event_id = event_id,
                        user_id = entry.user_id,
                        hours_awarded = entry.hours_awarded
                    ) TO db.session
            ELSE:
                IF entry.user_id IN existing:
                    DELETE existing[entry.user_id] FROM db.session

        # Full-replacement semantics: a member omitted from the payload is absent.
        FOR EACH old_user_id IN existing:
            IF old_user_id NOT IN submitted_ids:
                DELETE existing[old_user_id] FROM db.session

        COMMIT db.session
    EXCEPT database error:
        ROLLBACK db.session
        RETURN json_error("Could not save attendance", 500)

    saved_count = COUNT entries IN validated WHERE hours_awarded > 0
    RETURN 200 JSON { "saved": saved_count }

ROUTE GET "/users/<int:user_id>/hours" (REQUIRE valid JWT):
    auth_result = require_self_or_leader(user_id)
    IF auth_result IS an error response: RETURN auth_result

    user = db.session.get(User, user_id)
    IF user does not exist: RETURN json_error("User not found", 404)

    total = SCALAR(
        SELECT coalesce(sum(Attendance.hours_awarded), 0)
        WHERE Attendance.user_id == user_id
    )

    rows = RESULT OF
        SELECT Attendance.hours_awarded, Event.id, Event.name, Event.date
        JOIN Event ON Attendance.event_id == Event.id
        WHERE Attendance.user_id == user_id
        ORDER BY Event.date

    entries = [
        {
            event_id, event_name, date = ISO string, hours_awarded = float(...)
        }
        FOR EACH row
    ]

    RETURN 200 JSON {
        user_id = user_id,
        total_hours = float(total),
        entries = entries
    }
```

**Learn:** [SQLAlchemy 2.0 SELECT / `func.sum`](https://docs.sqlalchemy.org/en/20/core/selectable.html), [SQLAlchemy joins](https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html#joins), [`datetime.date.fromisoformat`](https://docs.python.org/3/library/datetime.html#datetime.date.fromisoformat), [Flask request JSON](https://flask.palletsprojects.com/en/stable/api/#flask.Request.get_json), [SQLAlchemy transactions / session](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html).

---

## S1. Supporting backend test & CI changes

These are not listed in the outline, but B3 deletes `/api/hello`, so the existing test and CI job must change.

### S1.1 `.github/workflows/ci.yml` — direct code

**Where:** replace the whole file. The backend job gets SQLite/JWT environment variables, so CI does not need a Postgres service. The frontend job is unchanged.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    env:
      DATABASE_URL: sqlite:///test.db
      JWT_SECRET_KEY: test-only-secret
      FRONTEND_ORIGIN: "*"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: backend/requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: ruff check .
      - run: ruff format --check .
      - run: pytest

  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run format:check
      - run: npm run build
```

### S1.2 `backend/tests/conftest.py` — NEW FILE, detailed pseudocode

**Where:** create this file. `TestConfig` uses `sqlite:///test.db`; each test gets a fresh schema and two seeded users.

```text
IMPORT pytest
IMPORT create_app FROM app
IMPORT TestConfig FROM app.config
IMPORT db FROM app.extensions
IMPORT User, Club FROM app.models
IMPORT generate_password_hash FROM werkzeug.security

FIXTURE app():
    app = create_app(TestConfig)

    WITH app.app_context():
        db.drop_all()
        db.create_all()

        leader = User(
            name="Test Leader", email="leader@test.local",
            password_hash=generate_password_hash("leader123"), is_leader=True
        )
        member = User(
            name="Test Member", email="member@test.local",
            password_hash=generate_password_hash("member123"), is_leader=False
        )
        club = Club(name="Robotics", description="Builds robots", leader_id=1)
        APPEND leader AND member TO club.members
        ADD leader, member, club TO db.session
        COMMIT

    YIELD app

    WITH app.app_context():
        db.session.remove()
        db.drop_all()

FIXTURE client(app):
    RETURN app.test_client()

HELPER FUNCTION auth_header(client, email, password):
    response = client.post("/api/auth/login", json={email, password})
    token = response.get_json()["access_token"]
    RETURN { "Authorization": "Bearer " + token }

FIXTURE leader_headers(client):
    RETURN auth_header(client, "leader@test.local", "leader123")

FIXTURE member_headers(client):
    RETURN auth_header(client, "member@test.local", "member123")
```

**Learn:** [Flask testing](https://flask.palletsprojects.com/en/stable/testing/), [pytest fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html), [Flask test client](https://flask.palletsprojects.com/en/stable/api/#flask.Flask.test_client).

### S1.3 `backend/tests/test_routes.py` — detailed pseudocode

**Where:** replace the whole file. Remove `test_hello`; keep `test_health`; add the cases below.

```text
TEST health:
    response = client.get("/api/health")
    ASSERT status 200 AND response.get_json()["status"] == "ok"

TEST login_success:
    response = client.post("/api/auth/login", json={
        email="leader@test.local", password="leader123"
    })
    ASSERT status 200
    ASSERT "access_token" IN response JSON
    ASSERT response JSON user.is_leader IS True
    ASSERT "password_hash" NOT IN response JSON user

TEST login_wrong_password:
    ASSERT status 401

TEST clubs_require_jwt:
    response = client.get("/api/clubs")
    ASSERT status 401

TEST member_cannot_create_club:
    response = client.post("/api/clubs", headers=member_headers, json={name:"Chess"})
    ASSERT status 403

TEST leader_can_create_club:
    response = client.post("/api/clubs", headers=leader_headers, json={
        name:"Chess", description:"Board games"
    })
    ASSERT status 201 AND response JSON leader_id == seeded leader id

TEST add_member_and_list_members:
    POST /api/clubs/1/members WITH leader_headers AND json {user_id: 2}
    ASSERT status 201 OR 409 (member was seeded)
    GET /api/clubs/1/members WITH leader_headers
    ASSERT status 200 AND seeded member email appears

TEST leader_of_other_club_is_forbidden:
    CREATE a second leader and a second club in the test
    TRY POST /api/clubs/1/members WITH second leader's token
    ASSERT status 403

TEST create_event_and_save_attendance:
    POST /api/clubs/1/events WITH leader_headers AND
         json {name:"First meeting", date:"2026-11-18", hours_worth:2}
    ASSERT status 201
    POST /api/events/1/attendance WITH leader_headers AND
         json {attendance:[{user_id:1,hours_awarded:2},{user_id:2,hours_awarded:0}]}
    ASSERT status 200 AND response JSON saved == 1
    GET /api/users/1/hours WITH leader_headers
    ASSERT status 200 AND response JSON total_hours == 2

TEST member_can_view_own_hours_but_not_another_users:
    GET /api/users/2/hours WITH member_headers -> 200
    GET /api/users/1/hours WITH member_headers -> 403
```

**Learn:** [pytest assertions](https://docs.pytest.org/en/stable/how-to/assertions.html), [Flask tutorial: tests](https://flask.palletsprojects.com/en/stable/tutorial/tests/), [Flask test client](https://flask.palletsprojects.com/en/stable/testing/).

---

# Frontend tasks

Create the new folders once:

```bash
cd frontend
mkdir -p src/context src/pages src/components
```

## Task F1 — Routing, state & auth flow

### F1.1 `frontend/package.json` — direct code / command

**Where:** run the command from `frontend/`. It edits `package.json` **and** `package-lock.json`; do not hand-edit the lockfile. `7.18.5` is the current `react-router-dom` release and supports React 19.

```bash
npm install react-router-dom@7.18.5
```

Expected `dependencies` block after the command:

```json
{
  "dependencies": {
    "react": "^19.2.8",
    "react-dom": "^19.2.8",
    "react-router-dom": "^7.18.5"
  }
}
```

### F1.2 `frontend/src/main.tsx` — direct code

**Where:** replace the whole file. `<BrowserRouter>` must be outside `<AuthProvider>` so context consumers can use router hooks.

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import './index.css'
import App from './App.tsx'
import { AuthProvider } from './context/AuthContext.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </StrictMode>,
)
```

### F1.3 `frontend/src/api.ts` — direct code

**Where:** replace the whole file. This deletes `getHello()` and adds `apiFetch()`, `apiGet()`, `apiPost()`, token helpers, every response interface, and every endpoint function used by F2–F5.

```ts
/**
 * Typed wrapper around fetch for the ClubTracker Flask API.
 *
 * Vite proxies /api to Flask in development and nginx does the same in
 * production, so the default BASE_URL is just "/api".
 */
const BASE_URL = import.meta.env.VITE_API_URL ?? '/api'
const TOKEN_KEY = 'clubtracker_token'

export function getToken(): string | null {
  if (typeof window === 'undefined') return null
  return window.localStorage.getItem(TOKEN_KEY)
}

export function setToken(token: string): void {
  if (typeof window !== 'undefined') window.localStorage.setItem(TOKEN_KEY, token)
}

export function clearToken(): void {
  if (typeof window !== 'undefined') window.localStorage.removeItem(TOKEN_KEY)
}

export interface User {
  id: number
  name: string
  email: string
  is_leader: boolean
}

export interface Club {
  id: number
  name: string
  description: string | null
  leader_id: number | null
}

export interface ClubEvent {
  id: number
  club_id: number
  name: string
  date: string
  hours_worth: number
}

export interface ClubMember {
  user_id: number
  club_id: number
}

export interface AttendanceRecord {
  id: number
  event_id: number
  user_id: number
  hours_awarded: number
}

export interface HoursEntry {
  event_id: number
  event_name: string
  date: string
  hours_awarded: number
}

export interface HoursSummary {
  user_id: number
  total_hours: number
  entries: HoursEntry[]
}

export interface LoginRequest {
  email: string
  password: string
}

export interface LoginResponse {
  access_token: string
  user: User
}

export interface CreateClubRequest {
  name: string
  description: string
}

export interface AddMemberRequest {
  user_id: number
}

export interface CreateEventRequest {
  name: string
  date: string
  hours_worth: number
}

export interface AttendanceEntry {
  user_id: number
  hours_awarded: number
}

export interface SaveAttendanceRequest {
  attendance: AttendanceEntry[]
}

export interface AttendanceSaveResponse {
  saved: number
}

interface ErrorBody {
  error?: string
  msg?: string
}

async function readError(response: Response): Promise<string> {
  try {
    const body = (await response.json()) as ErrorBody
    return body.error ?? body.msg ?? `Request failed with status ${response.status}`
  } catch {
    return `Request failed with status ${response.status}`
  }
}

export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
  const headers = new Headers(init.headers ?? {})
  const token = getToken()

  if (token) headers.set('Authorization', `Bearer ${token}`)
  if (init.body !== undefined && !headers.has('Content-Type')) {
    headers.set('Content-Type', 'application/json')
  }

  const response = await fetch(`${BASE_URL}${path}`, { ...init, headers })

  if (!response.ok) {
    if (response.status === 401) clearToken()
    throw new Error(await readError(response))
  }

  if (response.status === 204) return undefined as T
  return (await response.json()) as T
}

export function apiGet<T>(path: string): Promise<T> {
  return apiFetch<T>(path, { method: 'GET' })
}

export function apiPost<T, B>(path: string, body: B): Promise<T> {
  return apiFetch<T>(path, {
    method: 'POST',
    body: JSON.stringify(body),
  })
}

export function login(email: string, password: string): Promise<LoginResponse> {
  return apiPost<LoginResponse, LoginRequest>('/auth/login', { email, password })
}

export function getMe(): Promise<User> {
  return apiGet<User>('/auth/me')
}

export function getUsers(): Promise<User[]> {
  return apiGet<User[]>('/users')
}

export function getClubs(): Promise<Club[]> {
  return apiGet<Club[]>('/clubs')
}

export function getMyClubs(): Promise<Club[]> {
  return apiGet<Club[]>('/me/clubs')
}

export function getClub(clubId: number): Promise<Club> {
  return apiGet<Club>(`/clubs/${clubId}`)
}

export function createClub(input: CreateClubRequest): Promise<Club> {
  return apiPost<Club, CreateClubRequest>('/clubs', input)
}

export function getClubMembers(clubId: number): Promise<User[]> {
  return apiGet<User[]>(`/clubs/${clubId}/members`)
}

export function addClubMember(clubId: number, userId: number): Promise<ClubMember> {
  return apiPost<ClubMember, AddMemberRequest>(`/clubs/${clubId}/members`, { user_id: userId })
}

export function getClubEvents(clubId: number): Promise<ClubEvent[]> {
  return apiGet<ClubEvent[]>(`/clubs/${clubId}/events`)
}

export function createEvent(clubId: number, input: CreateEventRequest): Promise<ClubEvent> {
  return apiPost<ClubEvent, CreateEventRequest>(`/clubs/${clubId}/events`, input)
}

export function getEvent(eventId: number): Promise<ClubEvent> {
  return apiGet<ClubEvent>(`/events/${eventId}`)
}

export function getEventAttendance(eventId: number): Promise<AttendanceRecord[]> {
  return apiGet<AttendanceRecord[]>(`/events/${eventId}/attendance`)
}

export function saveAttendance(
  eventId: number,
  attendance: AttendanceEntry[],
): Promise<AttendanceSaveResponse> {
  return apiPost<AttendanceSaveResponse, SaveAttendanceRequest>(`/events/${eventId}/attendance`, {
    attendance,
  })
}

export function getHours(userId: number): Promise<HoursSummary> {
  return apiGet<HoursSummary>(`/users/${userId}/hours`)
}
```

Key behavior: `apiFetch()` reads `clubtracker_token` from `localStorage`, sets `Authorization: Bearer <token>` when present, sets `Content-Type: application/json` for bodies, clears the token on `401`, and throws a readable error from `{ error }` or `{ msg }`.

### F1.4 `frontend/src/context/AuthContext.tsx` — NEW FILE, direct code

**Where:** create this file. It stores `user`, derives `isLeader`, exposes `login()` / `logout()`, and restores the session with `GET /api/auth/me` when a token already exists. The ESLint disable comment is needed because this file exports both a component and a hook/context.

```tsx
/* eslint-disable react-refresh/only-export-components */
import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
  type ReactNode,
} from 'react'
import { clearToken, getMe, getToken, login as loginRequest, setToken } from '../api'
import type { User } from '../api'

export interface AuthValue {
  user: User | null
  isLeader: boolean
  isLoading: boolean
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

export const AuthContext = createContext<AuthValue | undefined>(undefined)

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    let active = true

    async function loadUser() {
      if (!getToken()) {
        if (active) setIsLoading(false)
        return
      }

      try {
        const profile = await getMe()
        if (active) setUser(profile)
      } catch {
        clearToken()
        if (active) setUser(null)
      } finally {
        if (active) setIsLoading(false)
      }
    }

    void loadUser()
    return () => {
      active = false
    }
  }, [])

  const login = useCallback(async (email: string, password: string) => {
    const result = await loginRequest(email, password)
    setToken(result.access_token)
    setUser(result.user)
    setIsLoading(false)
  }, [])

  const logout = useCallback(() => {
    clearToken()
    setUser(null)
    setIsLoading(false)
  }, [])

  const value = useMemo<AuthValue>(
    () => ({
      user,
      isLeader: user?.is_leader ?? false,
      isLoading,
      login,
      logout,
    }),
    [user, isLoading, login, logout],
  )

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}

export function useAuth(): AuthValue {
  const context = useContext(AuthContext)
  if (context === undefined) {
    throw new Error('useAuth must be used inside <AuthProvider>')
  }
  return context
}
```

### F1.5 No-change files

| File | Why no edit is needed |
| --- | --- |
| `frontend/src/vite-env.d.ts` | Already declares `VITE_API_URL?` |
| `frontend/vite.config.ts` | Already proxies `/api` to `http://localhost:5000` |
| `frontend/nginx.conf` | Already proxies `/api/` to `http://backend:5000/api/` and supports SPA fallback |

**Learn:** [React Router v7.18.5 routing](https://reactrouter.com/7.18.5/start/declarative/routing), [React Context: passing data deeply](https://react.dev/learn/passing-data-deeply-with-context), [`createContext`](https://react.dev/reference/react/createContext), [MDN Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch), [MDN `Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers), [MDN `localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), [TypeScript generics](https://www.typescriptlang.org/docs/handbook/2/generics.html), [Vite `server.proxy`](https://vite.dev/config/server-options.html#server-proxy).

## Task F2 — Login UI & protected routes

### F2.1 `frontend/src/pages/Login.tsx` — NEW FILE, direct code

**Where:** create this file. It is a controlled form; on submit it calls `login()` from `AuthContext`, which saves the token and user, then navigates to `/dashboard`.

```tsx
import { useState, type FormEvent } from 'react'
import { useNavigate } from 'react-router-dom'
import { useAuth } from '../context/AuthContext.tsx'

export default function Login() {
  const { login } = useAuth()
  const navigate = useNavigate()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [isSubmitting, setIsSubmitting] = useState(false)

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault()
    setError('')
    setIsSubmitting(true)

    try {
      await login(email, password)
      navigate('/dashboard', { replace: true })
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Could not sign in')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <main className="login-page">
      <form className="card stack" onSubmit={handleSubmit}>
        <h1>ClubTracker</h1>
        <p className="muted">Sign in to view clubs and honors society hours.</p>

        <label className="field">
          Email
          <input
            type="email"
            value={email}
            onChange={(event) => setEmail(event.target.value)}
            autoComplete="email"
            required
          />
        </label>

        <label className="field">
          Password
          <input
            type="password"
            value={password}
            onChange={(event) => setPassword(event.target.value)}
            autoComplete="current-password"
            required
          />
        </label>

        {error && <p className="error">{error}</p>}

        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Signing in…' : 'Sign in'}
        </button>
      </form>
    </main>
  )
}
```

### F2.2 `frontend/src/components/ProtectedRoute.tsx` — NEW FILE, direct code

**Where:** create this file. While `AuthContext` is restoring a session it shows a loading message; otherwise it redirects unauthenticated users to `/login`.

```tsx
import type { ReactNode } from 'react'
import { Navigate } from 'react-router-dom'
import { useAuth } from '../context/AuthContext.tsx'

interface ProtectedRouteProps {
  children: ReactNode
}

export default function ProtectedRoute({ children }: ProtectedRouteProps) {
  const { user, isLoading } = useAuth()

  if (isLoading) return <p className="page-message">Loading…</p>
  if (!user) return <Navigate to="/login" replace />

  return <>{children}</>
}
```

### F2.3 `frontend/src/App.tsx` — direct code

**Where:** replace the whole file. This clears the boilerplate greeting, defines `/login` as public, wraps `/`, `/clubs`, `/dashboard`, `/clubs/:clubId`, and `/events/:eventId/attendance` in `ProtectedPage`, and provides a shared header with logout.

**Important:** this single replacement already includes the `/dashboard` route requested by F3, the `/clubs/:clubId` route requested by F4, and the `/events/:eventId/attendance` route requested by F5. F3/F4/F5 below show the exact route lines for incremental application, but you should not add them twice.

```tsx
import type { ReactNode } from 'react'
import { Link, Navigate, Route, Routes, useNavigate } from 'react-router-dom'
import ProtectedRoute from './components/ProtectedRoute.tsx'
import { useAuth } from './context/AuthContext.tsx'
import AttendanceTracker from './pages/AttendanceTracker.tsx'
import ClubView from './pages/ClubView.tsx'
import Dashboard from './pages/Dashboard.tsx'
import Login from './pages/Login.tsx'

function Layout({ children }: { children: ReactNode }) {
  const { user, logout } = useAuth()
  const navigate = useNavigate()

  function handleLogout() {
    logout()
    navigate('/login', { replace: true })
  }

  return (
    <div className="app-shell">
      <header className="top-bar">
        <Link to="/dashboard" className="brand">
          ClubTracker
        </Link>
        <nav className="nav">
          <Link to="/dashboard">Dashboard</Link>
          {user && (
            <span className="muted">
              {user.name}
              {user.is_leader ? ' · Leader' : ''}
            </span>
          )}
          <button type="button" className="button-secondary" onClick={handleLogout}>
            Log out
          </button>
        </nav>
      </header>
      <main className="page">{children}</main>
    </div>
  )
}

function ProtectedPage({ children }: { children: ReactNode }) {
  return (
    <ProtectedRoute>
      <Layout>{children}</Layout>
    </ProtectedRoute>
  )
}

function App() {
  const { user } = useAuth()

  return (
    <Routes>
      <Route path="/login" element={user ? <Navigate to="/dashboard" replace /> : <Login />} />
      <Route
        path="/"
        element={
          <ProtectedPage>
            <Navigate to="/dashboard" replace />
          </ProtectedPage>
        }
      />
      <Route
        path="/dashboard"
        element={
          <ProtectedPage>
            <Dashboard />
          </ProtectedPage>
        }
      />
      <Route
        path="/clubs"
        element={
          <ProtectedPage>
            <Dashboard />
          </ProtectedPage>
        }
      />
      <Route
        path="/clubs/:clubId"
        element={
          <ProtectedPage>
            <ClubView />
          </ProtectedPage>
        }
      />
      <Route
        path="/events/:eventId/attendance"
        element={
          <ProtectedPage>
            <AttendanceTracker />
          </ProtectedPage>
        }
      />
      <Route path="*" element={<Navigate to={user ? '/dashboard' : '/login'} replace />} />
    </Routes>
  )
}

export default App
```

### F2.4 `frontend/index.html` — direct code

**Where:** replace `<title>Full Stack Example</title>` with `<title>ClubTracker</title>`. Full file shown for clarity.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ClubTracker</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**Learn:** [React Router `Navigate` / `Routes`](https://reactrouter.com/7.18.5/start/declarative/routing), [React Router `useNavigate`](https://api.reactrouter.com/v7/functions/react-router.useNavigate.html), [React forms](https://react.dev/reference/react-dom/components/input), [TypeScript `FormEvent`](https://react.dev/learn/typescript).

## Task F3 — Leader dashboard & club management UI

### F3.1 `frontend/src/pages/Dashboard.tsx` — NEW FILE, direct code

**Where:** create this file. It calls `GET /api/me/clubs`, shows clubs the user belongs to or leads, renders `HoursSummary`, and shows a “Create club” button only to leaders. It imports `HoursSummary` from F5.3, so create all files before running the build.

```tsx
import { useEffect, useState } from 'react'
import { Link } from 'react-router-dom'
import { getMyClubs, type Club } from '../api'
import CreateClubModal from '../components/CreateClubModal.tsx'
import HoursSummary from '../components/HoursSummary.tsx'
import { useAuth } from '../context/AuthContext.tsx'

export default function Dashboard() {
  const { user, isLeader } = useAuth()
  const [clubs, setClubs] = useState<Club[]>([])
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState('')
  const [showCreate, setShowCreate] = useState(false)

  useEffect(() => {
    let active = true

    async function loadClubs() {
      try {
        const data = await getMyClubs()
        if (active) setClubs(data)
      } catch (err) {
        if (active) setError(err instanceof Error ? err.message : 'Could not load clubs')
      } finally {
        if (active) setIsLoading(false)
      }
    }

    void loadClubs()
    return () => {
      active = false
    }
  }, [])

  if (!user) return null

  return (
    <div className="stack">
      <div className="spread">
        <div>
          <h1>Welcome, {user.name}</h1>
          <p className="muted">Clubs you lead or belong to.</p>
        </div>
        {isLeader && (
          <button type="button" onClick={() => setShowCreate(true)}>
            + Create club
          </button>
        )}
      </div>

      <HoursSummary userId={user.id} />

      {isLoading && <p className="page-message">Loading clubs…</p>}
      {error && <p className="error">{error}</p>}

      {!isLoading && !error && clubs.length === 0 && (
        <p className="page-message">You are not in any clubs yet.</p>
      )}

      <div className="club-grid">
        {clubs.map((club) => (
          <Link key={club.id} to={`/clubs/${club.id}`} className="club-card card">
            <h2>{club.name}</h2>
            <p className="muted">{club.description || 'No description yet.'}</p>
            {club.leader_id === user.id && <span className="pill">You lead this club</span>}
          </Link>
        ))}
      </div>

      {showCreate && (
        <CreateClubModal
          onClose={() => setShowCreate(false)}
          onCreated={(club) => {
            setClubs((current) => [...current, club])
            setShowCreate(false)
          }}
        />
      )}
    </div>
  )
}
```

### F3.2 `frontend/src/components/CreateClubModal.tsx` — NEW FILE, direct code

**Where:** create this file. The component itself returns `null` when `isLeader` is false, and submits `POST /api/clubs`.

```tsx
import { useState, type FormEvent } from 'react'
import { createClub, type Club } from '../api'
import { useAuth } from '../context/AuthContext.tsx'

interface CreateClubModalProps {
  onClose: () => void
  onCreated: (club: Club) => void
}

export default function CreateClubModal({ onClose, onCreated }: CreateClubModalProps) {
  const { isLeader } = useAuth()
  const [name, setName] = useState('')
  const [description, setDescription] = useState('')
  const [error, setError] = useState('')
  const [isSubmitting, setIsSubmitting] = useState(false)

  if (!isLeader) return null

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault()
    setError('')
    setIsSubmitting(true)

    try {
      const club = await createClub({ name, description })
      onCreated(club)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Could not create club')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <div className="modal-backdrop" role="presentation" onClick={onClose}>
      <section
        className="modal card stack"
        role="dialog"
        aria-modal="true"
        aria-labelledby="create-club-title"
        onClick={(event) => event.stopPropagation()}
      >
        <h2 id="create-club-title">Create a club</h2>

        <form className="stack" onSubmit={handleSubmit}>
          <label className="field">
            Club name
            <input value={name} onChange={(event) => setName(event.target.value)} required />
          </label>

          <label className="field">
            Description
            <textarea
              value={description}
              onChange={(event) => setDescription(event.target.value)}
              rows={3}
            />
          </label>

          {error && <p className="error">{error}</p>}

          <div className="toolbar">
            <button type="button" className="button-secondary" onClick={onClose}>
              Cancel
            </button>
            <button type="submit" disabled={isSubmitting}>
              {isSubmitting ? 'Creating…' : 'Create club'}
            </button>
          </div>
        </form>
      </section>
    </div>
  )
}
```

### F3.3 `frontend/src/App.tsx` — `/dashboard` route, direct code

**Where:** inside `<Routes>`, after the `/login` route. Already present in the F2.3 full replacement.

```tsx
<Route
  path="/dashboard"
  element={
    <ProtectedPage>
      <Dashboard />
    </ProtectedPage>
  }
/>
```

**Learn:** [React `useEffect`](https://react.dev/reference/react/useEffect), [React `useState`](https://react.dev/reference/react/useState), [Conditional rendering](https://react.dev/learn/conditional-rendering), [MDN `<dialog>`/modal accessibility patterns](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog).

## Task F4 — Member directory & adding members

### F4.1 `frontend/src/pages/ClubView.tsx` — NEW FILE, direct code

**Where:** create this file. It reads `clubId` from `useParams()`, fetches the club and its members, conditionally renders `AddMemberForm` for the club’s actual leader, and embeds `EventList` from F5.1.

```tsx
import { useEffect, useState } from 'react'
import { Link, useParams } from 'react-router-dom'
import { getClub, getClubMembers, type Club, type User } from '../api'
import AddMemberForm from '../components/AddMemberForm.tsx'
import EventList from '../components/EventList.tsx'
import { useAuth } from '../context/AuthContext.tsx'

export default function ClubView() {
  const { clubId: clubIdParam } = useParams()
  const clubId = Number(clubIdParam)
  const isValidClubId = Number.isInteger(clubId) && clubId > 0
  const { user, isLeader } = useAuth()
  const [club, setClub] = useState<Club | null>(null)
  const [members, setMembers] = useState<User[]>([])
  const [isLoading, setIsLoading] = useState(isValidClubId)
  const [error, setError] = useState(isValidClubId ? '' : 'Invalid club id')

  useEffect(() => {
    if (!isValidClubId) return
    let active = true

    async function loadClub() {
      try {
        const [clubData, memberData] = await Promise.all([getClub(clubId), getClubMembers(clubId)])
        if (!active) return
        setClub(clubData)
        setMembers(memberData)
      } catch (err) {
        if (active) setError(err instanceof Error ? err.message : 'Could not load club')
      } finally {
        if (active) setIsLoading(false)
      }
    }

    void loadClub()
    return () => {
      active = false
    }
  }, [clubId, isValidClubId])

  if (isLoading) return <p className="page-message">Loading club…</p>
  if (error) return <p className="error">{error}</p>
  if (!club) return <p className="page-message">Club not found.</p>

  const canManage = isLeader && user?.id === club.leader_id

  return (
    <div className="stack">
      <Link to="/dashboard" className="back-link">
        ← Back to dashboard
      </Link>

      <section className="card">
        <h1>{club.name}</h1>
        <p className="muted">{club.description || 'No description yet.'}</p>
      </section>

      <section className="card">
        <h2>Members ({members.length})</h2>
        {members.length === 0 ? (
          <p className="muted">No members yet.</p>
        ) : (
          <ul className="plain-list">
            {members.map((member) => (
              <li key={member.id}>
                {member.name} <span className="muted">({member.email})</span>
              </li>
            ))}
          </ul>
        )}
        {canManage && (
          <AddMemberForm
            clubId={club.id}
            existingMemberIds={members.map((member) => member.id)}
            onMemberAdded={(member) =>
              setMembers((current) =>
                current.some((existing) => existing.id === member.id)
                  ? current
                  : [...current, member],
              )
            }
          />
        )}
      </section>

      <EventList clubId={club.id} isLeader={canManage} />
    </div>
  )
}
```

### F4.2 `frontend/src/components/AddMemberForm.tsx` — NEW FILE, direct code

**Where:** create this file. It fetches `GET /api/users`, filters out existing members, and posts `{ user_id }` to `POST /api/clubs/<club_id>/members`.

```tsx
import { useEffect, useState, type FormEvent } from 'react'
import { addClubMember, getUsers, type User } from '../api'
import { useAuth } from '../context/AuthContext.tsx'

interface AddMemberFormProps {
  clubId: number
  existingMemberIds: number[]
  onMemberAdded: (user: User) => void
}

export default function AddMemberForm({
  clubId,
  existingMemberIds,
  onMemberAdded,
}: AddMemberFormProps) {
  const { isLeader } = useAuth()
  const [users, setUsers] = useState<User[]>([])
  const [selectedUserId, setSelectedUserId] = useState('')
  const [error, setError] = useState('')
  const [isSubmitting, setIsSubmitting] = useState(false)

  useEffect(() => {
    let active = true

    async function loadUsers() {
      try {
        const data = await getUsers()
        if (active) setUsers(data)
      } catch (err) {
        if (active) setError(err instanceof Error ? err.message : 'Could not load students')
      }
    }

    void loadUsers()
    return () => {
      active = false
    }
  }, [])

  if (!isLeader) return null

  const availableUsers = users.filter((user) => !existingMemberIds.includes(user.id))

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault()
    setError('')
    const userId = Number(selectedUserId)
    const selectedUser = users.find((user) => user.id === userId)

    if (!selectedUser) {
      setError('Choose a student first')
      return
    }

    setIsSubmitting(true)
    try {
      await addClubMember(clubId, userId)
      onMemberAdded(selectedUser)
      setSelectedUserId('')
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Could not add member')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <form className="toolbar" onSubmit={handleSubmit}>
      <label className="field">
        Add student
        <select
          value={selectedUserId}
          onChange={(event) => setSelectedUserId(event.target.value)}
          required
        >
          <option value="">Select a student…</option>
          {availableUsers.map((user) => (
            <option key={user.id} value={user.id}>
              {user.name} ({user.email})
            </option>
          ))}
        </select>
      </label>
      <button type="submit" disabled={isSubmitting || availableUsers.length === 0}>
        {isSubmitting ? 'Adding…' : 'Add member'}
      </button>
      {error && <p className="error">{error}</p>}
    </form>
  )
}
```

### F4.3 `frontend/src/App.tsx` — `/clubs/:clubId` route, direct code

**Where:** inside `<Routes>`, after `/clubs`. Already present in the F2.3 full replacement.

```tsx
<Route
  path="/clubs/:clubId"
  element={
    <ProtectedPage>
      <ClubView />
    </ProtectedPage>
  }
/>
```

**Learn:** [React Router dynamic segments / `useParams`](https://reactrouter.com/7.18.5/start/declarative/routing#dynamic-segments), [React lists and keys](https://react.dev/learn/rendering-lists), [MDN `<select>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/select), [MDN `Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all).

## Task F5 — Attendance & hours tracker UI

### F5.1 `frontend/src/components/EventList.tsx` — NEW FILE, direct code

**Where:** create this file and render it from `ClubView.tsx` (already done in F4.1). It fetches `GET /api/clubs/<id>/events`, splits events into upcoming/past, shows a leader-only “+ Create event” button, posts `POST /api/clubs/<id>/events`, and links each event to its attendance page.

```tsx
import { useEffect, useMemo, useState, type FormEvent } from 'react'
import { Link } from 'react-router-dom'
import { createEvent, getClubEvents, type ClubEvent, type CreateEventRequest } from '../api'

interface EventListProps {
  clubId: number
  isLeader: boolean
}

function EventRow({ event }: { event: ClubEvent }) {
  return (
    <li className="event-row spread">
      <span>
        <strong>{event.name}</strong> · {event.date} · {event.hours_worth.toFixed(1)} h
      </span>
      <Link to={`/events/${event.id}/attendance`}>Open attendance →</Link>
    </li>
  )
}

export default function EventList({ clubId, isLeader }: EventListProps) {
  const [events, setEvents] = useState<ClubEvent[]>([])
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState('')
  const [showForm, setShowForm] = useState(false)
  const [name, setName] = useState('')
  const [date, setDate] = useState('')
  const [hoursWorth, setHoursWorth] = useState('1')
  const [isSubmitting, setIsSubmitting] = useState(false)

  useEffect(() => {
    let active = true

    async function loadEvents() {
      try {
        const data = await getClubEvents(clubId)
        if (active) setEvents(data)
      } catch (err) {
        if (active) setError(err instanceof Error ? err.message : 'Could not load events')
      } finally {
        if (active) setIsLoading(false)
      }
    }

    void loadEvents()
    return () => {
      active = false
    }
  }, [clubId])

  const today = new Date().toISOString().slice(0, 10)
  const { upcoming, past } = useMemo(() => {
    const sorted = [...events].sort((a, b) => a.date.localeCompare(b.date))
    return {
      upcoming: sorted.filter((event) => event.date >= today),
      past: sorted.filter((event) => event.date < today).reverse(),
    }
  }, [events, today])

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault()
    setError('')
    setIsSubmitting(true)

    const payload: CreateEventRequest = {
      name,
      date,
      hours_worth: Number(hoursWorth),
    }

    try {
      const created = await createEvent(clubId, payload)
      setEvents((current) => [...current, created])
      setName('')
      setDate('')
      setHoursWorth('1')
      setShowForm(false)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Could not create event')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <section className="card stack">
      <div className="spread">
        <h2>Meetings</h2>
        {isLeader && (
          <button type="button" onClick={() => setShowForm((visible) => !visible)}>
            {showForm ? 'Hide form' : '+ Create event'}
          </button>
        )}
      </div>

      {showForm && isLeader && (
        <form className="form-grid" onSubmit={handleSubmit}>
          <label className="field">
            Name
            <input value={name} onChange={(event) => setName(event.target.value)} required />
          </label>
          <label className="field">
            Date
            <input
              type="date"
              value={date}
              onChange={(event) => setDate(event.target.value)}
              required
            />
          </label>
          <label className="field">
            Hours worth
            <input
              type="number"
              min="0"
              step="0.25"
              value={hoursWorth}
              onChange={(event) => setHoursWorth(event.target.value)}
              required
            />
          </label>
          <button type="submit" disabled={isSubmitting}>
            {isSubmitting ? 'Creating…' : 'Create event'}
          </button>
        </form>
      )}

      {isLoading && <p className="page-message">Loading meetings…</p>}
      {error && <p className="error">{error}</p>}

      {!isLoading && !error && (
        <div className="stack">
          <div className="event-section">
            <h3>Upcoming</h3>
            {upcoming.length === 0 ? (
              <p className="muted">No upcoming meetings.</p>
            ) : (
              <ul className="event-list">
                {upcoming.map((event) => (
                  <EventRow key={event.id} event={event} />
                ))}
              </ul>
            )}
          </div>

          <div className="event-section">
            <h3>Past</h3>
            {past.length === 0 ? (
              <p className="muted">No past meetings.</p>
            ) : (
              <ul className="event-list">
                {past.map((event) => (
                  <EventRow key={event.id} event={event} />
                ))}
              </ul>
            )}
          </div>
        </div>
      )}
    </section>
  )
}
```

### F5.2 `frontend/src/pages/AttendanceTracker.tsx` — NEW FILE, direct code

**Where:** create this file. It fetches the event, the club’s members, and existing attendance; renders one row per member with a Present checkbox and Hours input; and sends one `{ user_id, hours_awarded }` entry per member to `POST /api/events/<id>/attendance`. Absent rows send `0`, which the backend treats as “delete/clear”.

```tsx
import { useEffect, useState } from 'react'
import { Link, useParams } from 'react-router-dom'
import {
  getClubMembers,
  getEvent,
  getEventAttendance,
  saveAttendance,
  type ClubEvent,
  type User,
} from '../api'
import { useAuth } from '../context/AuthContext.tsx'

interface AttendanceRow {
  present: boolean
  hours: number
}

export default function AttendanceTracker() {
  const { eventId: eventIdParam } = useParams()
  const eventId = Number(eventIdParam)
  const isValidEventId = Number.isInteger(eventId) && eventId > 0
  const { isLeader } = useAuth()
  const [event, setEvent] = useState<ClubEvent | null>(null)
  const [members, setMembers] = useState<User[]>([])
  const [rows, setRows] = useState<Record<number, AttendanceRow>>({})
  const [isLoading, setIsLoading] = useState(isValidEventId)
  const [error, setError] = useState(isValidEventId ? '' : 'Invalid event id')
  const [isSaving, setIsSaving] = useState(false)
  const [success, setSuccess] = useState('')

  useEffect(() => {
    if (!isValidEventId) return
    let active = true

    async function loadTracker() {
      try {
        const eventData = await getEvent(eventId)
        const [memberData, attendanceData] = await Promise.all([
          getClubMembers(eventData.club_id),
          getEventAttendance(eventId),
        ])
        if (!active) return

        const saved = new Map(
          attendanceData.map((record) => [record.user_id, record.hours_awarded]),
        )
        const initialRows = Object.fromEntries(
          memberData.map((member) => [
            member.id,
            {
              present: saved.has(member.id),
              hours: saved.get(member.id) ?? 0,
            },
          ]),
        ) as Record<number, AttendanceRow>

        setEvent(eventData)
        setMembers(memberData)
        setRows(initialRows)
      } catch (err) {
        if (active) setError(err instanceof Error ? err.message : 'Could not load attendance')
      } finally {
        if (active) setIsLoading(false)
      }
    }

    void loadTracker()
    return () => {
      active = false
    }
  }, [eventId, isValidEventId])

  function updateRow(userId: number, patch: Partial<AttendanceRow>) {
    setRows((current) => ({
      ...current,
      [userId]: { ...(current[userId] ?? { present: false, hours: 0 }), ...patch },
    }))
  }

  async function handleSave() {
    if (!event || !isLeader) return
    setError('')
    setSuccess('')
    setIsSaving(true)

    try {
      const attendance = members.map((member) => ({
        user_id: member.id,
        hours_awarded: rows[member.id]?.present ? Number(rows[member.id]?.hours ?? 0) : 0,
      }))

      if (
        attendance.some(
          (entry) => entry.hours_awarded < 0 || entry.hours_awarded > event.hours_worth,
        )
      ) {
        throw new Error(`Hours must be between 0 and ${event.hours_worth}.`)
      }

      const result = await saveAttendance(event.id, attendance)
      setSuccess(`Saved ${result.saved} attendance record${result.saved === 1 ? '' : 's'}.`)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Could not save attendance')
    } finally {
      setIsSaving(false)
    }
  }

  if (isLoading) return <p className="page-message">Loading attendance…</p>
  if (error && !event) return <p className="error">{error}</p>
  if (!event) return <p className="page-message">Event not found.</p>

  return (
    <div className="stack">
      <Link to={`/clubs/${event.club_id}`} className="back-link">
        ← Back to club
      </Link>

      <section className="card">
        <h1>{event.name}</h1>
        <p className="muted">
          {event.date} · up to {event.hours_worth.toFixed(1)} hours per member
        </p>
      </section>

      <section className="card stack">
        <h2>Attendance</h2>

        {error && <p className="error">{error}</p>}
        {success && <p className="success">{success}</p>}

        <table className="table">
          <thead>
            <tr>
              <th>Member</th>
              <th>Present</th>
              <th>Hours</th>
            </tr>
          </thead>
          <tbody>
            {members.map((member) => {
              const row = rows[member.id] ?? { present: false, hours: 0 }
              return (
                <tr key={member.id}>
                  <td>
                    {member.name} <span className="muted">({member.email})</span>
                  </td>
                  <td>
                    <input
                      type="checkbox"
                      checked={row.present}
                      disabled={!isLeader}
                      onChange={(changeEvent) =>
                        updateRow(member.id, { present: changeEvent.target.checked })
                      }
                    />
                  </td>
                  <td>
                    <input
                      type="number"
                      min="0"
                      max={event.hours_worth}
                      step="0.25"
                      value={row.hours}
                      disabled={!isLeader || !row.present}
                      onChange={(changeEvent) =>
                        updateRow(member.id, { hours: Number(changeEvent.target.value) })
                      }
                    />
                  </td>
                </tr>
              )
            })}
          </tbody>
        </table>

        {isLeader && (
          <button type="button" onClick={handleSave} disabled={isSaving}>
            {isSaving ? 'Saving…' : 'Save attendance'}
          </button>
        )}
        {!isLeader && <p className="muted">Only club leaders can edit attendance.</p>}
      </section>
    </div>
  )
}
```

### F5.3 `frontend/src/components/HoursSummary.tsx` — NEW FILE, direct code

**Where:** create this file and render it from `Dashboard.tsx` (already done in F3.1). It calls `GET /api/users/<user_id>/hours`.

```tsx
import { useEffect, useState } from 'react'
import { getHours, type HoursSummary as HoursSummaryData } from '../api'

interface HoursSummaryProps {
  userId: number
}

export default function HoursSummary({ userId }: HoursSummaryProps) {
  const [summary, setSummary] = useState<HoursSummaryData | null>(null)
  const [error, setError] = useState('')

  useEffect(() => {
    let active = true

    async function loadHours() {
      try {
        const data = await getHours(userId)
        if (active) setSummary(data)
      } catch (err) {
        if (active) setError(err instanceof Error ? err.message : 'Could not load hours')
      }
    }

    void loadHours()
    return () => {
      active = false
    }
  }, [userId])

  if (error) {
    return (
      <section className="card">
        <h2>Honors hours</h2>
        <p className="error">{error}</p>
      </section>
    )
  }

  if (!summary) {
    return (
      <section className="card">
        <h2>Honors hours</h2>
        <p className="page-message">Loading hours…</p>
      </section>
    )
  }

  return (
    <section className="card">
      <h2>Honors hours</h2>
      <p className="hours-total">{summary.total_hours.toFixed(1)} total hours</p>
      {summary.entries.length === 0 ? (
        <p className="muted">No attendance recorded yet.</p>
      ) : (
        <ul className="plain-list">
          {summary.entries.map((entry) => (
            <li key={entry.event_id}>
              {entry.date} · {entry.event_name} · {entry.hours_awarded.toFixed(1)} h
            </li>
          ))}
        </ul>
      )}
    </section>
  )
}
```

### F5.4 `frontend/src/App.tsx` — attendance route, direct code

**Where:** inside `<Routes>`, after `/clubs/:clubId`. Already present in the F2.3 full replacement.

```tsx
<Route
  path="/events/:eventId/attendance"
  element={
    <ProtectedPage>
      <AttendanceTracker />
    </ProtectedPage>
  }
/>
```

### F5.5 `frontend/src/index.css` — direct code

**Where:** replace the whole file. This is the global stylesheet for F2–F5: app shell, cards, forms, tables, modal, club grid, event list, and responsive header.

```css
:root {
  font-family:
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    sans-serif;
  color-scheme: light;
  color: #1f2937;
  background: #f4f6fb;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  min-height: 100vh;
}

button,
input,
select,
textarea {
  font: inherit;
}

button {
  border: 0;
  border-radius: 8px;
  background: #4f46e5;
  color: white;
  cursor: pointer;
  padding: 0.65rem 1rem;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

button.button-secondary {
  background: #e5e7eb;
  color: #1f2937;
}

a {
  color: #4f46e5;
}

.app-shell {
  min-height: 100vh;
}

.top-bar {
  align-items: center;
  background: #111827;
  color: white;
  display: flex;
  gap: 1rem;
  justify-content: space-between;
  padding: 0.9rem 1.5rem;
}

.top-bar a {
  color: white;
  text-decoration: none;
}

.brand {
  font-size: 1.2rem;
  font-weight: 700;
}

.nav {
  align-items: center;
  display: flex;
  gap: 1rem;
}

.page {
  margin: 0 auto;
  max-width: 1000px;
  padding: 2rem 1.25rem 4rem;
}

.login-page {
  align-items: center;
  display: grid;
  min-height: 100vh;
  padding: 1.25rem;
  place-items: center;
}

.login-page .card {
  width: min(100%, 420px);
}

.card {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  box-shadow: 0 8px 24px rgb(15 23 42 / 6%);
  padding: 1.25rem;
}

.stack {
  display: grid;
  gap: 1rem;
}

.spread {
  align-items: center;
  display: flex;
  gap: 1rem;
  justify-content: space-between;
}

.toolbar {
  align-items: flex-end;
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.field {
  display: grid;
  gap: 0.35rem;
  font-weight: 600;
}

.field input,
.field select,
.field textarea,
.table input[type='number'] {
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 0.55rem 0.7rem;
  width: 100%;
}

.form-grid {
  align-items: end;
  display: grid;
  gap: 0.75rem;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
}

.club-grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
}

.club-card {
  color: inherit;
  display: block;
  text-decoration: none;
  transition: transform 120ms ease;
}

.club-card:hover {
  transform: translateY(-2px);
}

.club-card h2 {
  margin: 0 0 0.5rem;
}

.pill {
  background: #eef2ff;
  border-radius: 999px;
  color: #3730a3;
  display: inline-block;
  font-size: 0.8rem;
  margin-top: 0.5rem;
  padding: 0.2rem 0.6rem;
}

.muted {
  color: #6b7280;
}

.error {
  color: #b91c1c;
}

.success {
  color: #15803d;
}

.page-message {
  color: #6b7280;
}

.hours-total {
  font-size: 1.8rem;
  font-weight: 700;
  margin: 0;
}

.plain-list,
.event-list {
  display: grid;
  gap: 0.4rem;
  list-style: none;
  margin: 0;
  padding: 0;
}

.event-section h3 {
  margin-bottom: 0.5rem;
}

.event-row {
  border-bottom: 1px solid #e5e7eb;
  padding: 0.5rem 0;
}

.back-link {
  display: inline-block;
  text-decoration: none;
}

.modal-backdrop {
  align-items: center;
  background: rgb(15 23 42 / 55%);
  display: flex;
  inset: 0;
  justify-content: center;
  padding: 1rem;
  position: fixed;
  z-index: 10;
}

.modal {
  width: min(100%, 480px);
}

.table {
  border-collapse: collapse;
  width: 100%;
}

.table th,
.table td {
  border-bottom: 1px solid #e5e7eb;
  padding: 0.65rem;
  text-align: left;
}

.table input[type='number'] {
  max-width: 110px;
}

@media (max-width: 640px) {
  .top-bar,
  .spread {
    align-items: flex-start;
    flex-direction: column;
  }
}
```

The page uses the global `isLeader` flag to show edit controls; `leader_required` in `utils.py` still enforces the **specific club’s** leader on the server. To hide the Save button from leaders of other clubs too, add `club.leader_id` to the `GET /api/events/<id>` response and compare it with `user.id`.

**Learn:** [React controlled inputs](https://react.dev/reference/react-dom/components/input), [React `useMemo`](https://react.dev/reference/react/useMemo), [MDN `<table>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/table), [MDN `<input type="number">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/number), [MDN `<input type="date">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/date), [MDN `Object.fromEntries`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/fromEntries).

---

## 6. Run and verify everything

### Local development (two terminals or `make dev`)

```bash
# once, from repo root
cp .env.example .env
# edit JWT_SECRET_KEY and POSTGRES_PASSWORD

cd backend
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
.venv/bin/flask --app wsgi init-db
.venv/bin/python wsgi.py          # http://localhost:5000

cd ../frontend
npm install                       # installs react-router-dom too, after F1.1
npm run dev                       # http://localhost:5173
```

Then open `http://localhost:5173`. Sign in with:

| Role | Email | Password |
| --- | --- | --- |
| Leader | `leader@example.com` | `leader123` |
| Member | `member@example.com` | `member123` |

Expected flow:

1. Unauthenticated `/dashboard` redirects to `/login`.
2. Leader login shows Dashboard, HoursSummary, “Create club”, and seeded Robotics club.
3. Club view shows members, AddMemberForm, and EventList.
4. Create an event, open its attendance page, check members, enter hours, save.
5. The member’s HoursSummary increases; a member cannot edit attendance.

### Production-style Docker run

```bash
cp .env.example .env
# edit secrets
docker compose up --build -d
docker compose exec backend flask --app wsgi init-db
# open http://localhost:8080
```

`scripts/deploy.sh` and `.github/workflows/deploy.yml` do **not** need edits for this outline; run `init-db` once on the server after the first deploy.

### Checks before pushing

```bash
make format
make lint
make test

# or individually
cd backend && .venv/bin/pytest && .venv/bin/ruff check . && .venv/bin/ruff format --check .
cd frontend && npm run lint && npm run typecheck && npm run format:check && npm run build
```

---

## 7. Resource index

### Python / backend pseudocode

| Topic | Resource |
| --- | --- |
| App factory | [Flask Application Factories](https://flask.palletsprojects.com/en/stable/patterns/appfactories/) |
| Custom CLI commands | [Flask Command Line Interface](https://flask.palletsprojects.com/en/stable/cli/) |
| Config | [Flask configuration](https://flask.palletsprojects.com/en/stable/config/) · [Flask-SQLAlchemy config](https://flask-sqlalchemy.readthedocs.io/en/stable/config/) · [JWT options](https://flask-jwt-extended.readthedocs.io/en/stable/options.html) |
| ORM setup | [Flask-SQLAlchemy Quick Start](https://flask-sqlalchemy.readthedocs.io/en/stable/quickstart/) · [Models and Tables](https://flask-sqlalchemy.readthedocs.io/en/stable/models/) · [App context](https://flask-sqlalchemy.readthedocs.io/en/stable/contexts/) |
| Models/relationships | [SQLAlchemy relationships](https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html) · [Unique constraints](https://docs.sqlalchemy.org/en/20/core/constraints.html#unique-constraint) · [SELECT guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html) · [Transactions](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html) |
| Auth | [Flask-JWT-Extended Basic Usage](https://flask-jwt-extended.readthedocs.io/en/stable/basic_usage.html) · [Additional claims](https://flask-jwt-extended.readthedocs.io/en/stable/add_custom_data_claims.html) · [Custom decorators](https://flask-jwt-extended.readthedocs.io/en/stable/custom_decorators.html) · [API reference](https://flask-jwt-extended.readthedocs.io/en/stable/api.html) |
| Passwords | [Werkzeug security helpers](https://werkzeug.palletsprojects.com/en/stable/utils/) |
| Decorators / dates / env | [`functools.wraps`](https://docs.python.org/3/library/functools.html#functools.wraps) · [`date.fromisoformat`](https://docs.python.org/3/library/datetime.html#datetime.date.fromisoformat) · [`timedelta`](https://docs.python.org/3/library/datetime.html#timedelta-objects) · [`os.environ`](https://docs.python.org/3/library/os.html#os.environ) |
| Tests | [Flask testing](https://flask.palletsprojects.com/en/stable/testing/) · [Flask tutorial tests](https://flask.palletsprojects.com/en/stable/tutorial/tests/) · [pytest fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html) · [pytest assertions](https://docs.pytest.org/en/stable/how-to/assertions.html) |
| Migrations (next step) | [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/) · [Alembic](https://alembic.sqlalchemy.org/en/latest/) |
| Security | [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) · [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) |

### Frontend / tooling

| Topic | Resource |
| --- | --- |
| Routing | [React Router v7.18.5 Routing](https://reactrouter.com/7.18.5/start/declarative/routing) · [URL values / `useParams`](https://reactrouter.com/7.18.5/start/declarative/url-values) · [v7 API reference](https://api.reactrouter.com/v7/) |
| React state/context | [Passing data deeply with Context](https://react.dev/learn/passing-data-deeply-with-context) · [`createContext`](https://react.dev/reference/react/createContext) · [`useEffect`](https://react.dev/reference/react/useEffect) · [`useState`](https://react.dev/reference/react/useState) · [`useMemo`](https://react.dev/reference/react/useMemo) |
| Forms/tables | [React forms](https://react.dev/reference/react-dom/components/input) · [Lists and keys](https://react.dev/learn/rendering-lists) · [Conditional rendering](https://react.dev/learn/conditional-rendering) |
| HTTP | [MDN Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) · [`RequestInit`](https://developer.mozilla.org/en-US/docs/Web/API/RequestInit) · [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) · [`localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) |
| TypeScript | [Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) · [React + TypeScript](https://react.dev/learn/typescript) |
| Build/proxy | [Vite `server.proxy`](https://vite.dev/config/server-options.html#server-proxy) · [Vite config](https://vite.dev/config/) |
| Docker | [Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/) · [Postgres official image](https://hub.docker.com/_/postgres) · [Compose file reference](https://docs.docker.com/reference/compose-file/services/) |
| CI | [GitHub Actions workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) |

---

## 8. Production caveats (do not skip later)

1. **JWT in `localStorage`** is what the outline requests, but it is readable by any JavaScript on the page. For a real deployment, study HttpOnly `Secure` cookies, CSRF protection, and XSS prevention.
2. **`db.create_all()` is not a migration.** Once you change a model, add `Flask-Migrate`, run `flask db init`, `flask db migrate`, and `flask db upgrade`, and commit the `migrations/` folder.
3. **Replace all demo credentials and default secrets.** `JWT_SECRET_KEY`, `POSTGRES_PASSWORD`, and the seeded `leader123` / `member123` passwords are development-only.
4. **`FRONTEND_ORIGIN=*`** is acceptable locally behind the proxy; set it to the real origin if the frontend and API are ever hosted separately.
5. **`GET /api/users` exposes every user’s name/email to any logged-in member.** It is needed for the outline’s dropdown, but in production restrict it to leaders or add a search endpoint with minimal fields.
6. **Postgres host port** is bound to `127.0.0.1` in the supplied Compose file. Keep it that way unless a local database tool specifically needs direct access.
7. **Authorization is enforced server-side.** Hiding a button in React is only UX; `leader_required` and the ownership checks in `utils.py` are the actual security boundary.

---

## 9. File checklist

| Task | File | Action | Format in this guide |
| --- | --- | --- | --- |
| B1 | `backend/requirements.txt` | replace | direct code |
| B1 | `docker-compose.yml` | replace | direct code |
| B1 | `.env.example` | replace | direct code |
| B1 | `.gitignore` | add Python ignores | direct code |
| B1 | `backend/app/config.py` | replace class, add `TestConfig` | pseudocode |
| B1 | `backend/app/extensions.py` | **new** | pseudocode |
| B1 | `backend/app/__init__.py` | replace factory | pseudocode |
| B1 | `backend/app/models.py` | **new** | pseudocode |
| B2 | `backend/requirements.txt` | no second edit | note |
| B2 | `backend/app/config.py` | covered in B1.5 | note |
| B2 | `backend/app/__init__.py` | covered in B1.7 | note |
| B2 | `backend/app/auth.py` | **new** | pseudocode |
| B2 | `backend/app/utils.py` | **new** | pseudocode |
| B3 | `backend/app/routes.py` | replace `/hello`, add club/member routes | pseudocode |
| B4 | `backend/app/routes.py` | append event/attendance/hours routes | pseudocode |
| S1 | `.github/workflows/ci.yml` | replace | direct code |
| S1 | `backend/tests/conftest.py` | **new** | pseudocode |
| S1 | `backend/tests/test_routes.py` | replace | pseudocode |
| F1 | `frontend/package.json` + `package-lock.json` | `npm install` | command + JSON |
| F1 | `frontend/src/main.tsx` | replace | direct code |
| F1 | `frontend/src/api.ts` | replace | direct code |
| F1 | `frontend/src/context/AuthContext.tsx` | **new** | direct code |
| F2 | `frontend/src/pages/Login.tsx` | **new** | direct code |
| F2 | `frontend/src/components/ProtectedRoute.tsx` | **new** | direct code |
| F2 | `frontend/src/App.tsx` | replace (all routes) | direct code |
| F2 | `frontend/index.html` | change title | direct code |
| F3 | `frontend/src/pages/Dashboard.tsx` | **new** | direct code |
| F3 | `frontend/src/components/CreateClubModal.tsx` | **new** | direct code |
| F3 | `frontend/src/App.tsx` | `/dashboard` route | direct code |
| F4 | `frontend/src/pages/ClubView.tsx` | **new** | direct code |
| F4 | `frontend/src/components/AddMemberForm.tsx` | **new** | direct code |
| F4 | `frontend/src/App.tsx` | `/clubs/:clubId` route | direct code |
| F5 | `frontend/src/components/EventList.tsx` | **new** | direct code |
| F5 | `frontend/src/pages/AttendanceTracker.tsx` | **new** | direct code |
| F5 | `frontend/src/components/HoursSummary.tsx` | **new** | direct code |
| F5 | `frontend/src/App.tsx` | `/events/:eventId/attendance` route | direct code |
| F5 | `frontend/src/index.css` | replace | direct code |