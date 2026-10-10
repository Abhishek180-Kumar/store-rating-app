# Store Rating Platform

A full stack web application where users rate stores from **1 to 5**. A single
login system serves three roles — **System Administrator**, **Normal User** and
**Store Owner** — and the role attached to the account decides what the user can
see and do after logging in.

| Layer | Technology |
| --- | --- |
| Frontend | React 18 (Vite), React Router 6, hand written CSS |
| Backend | Node.js + Express 4 (ES modules) |
| Database | PostgreSQL 16 |
| Auth | JWT (`jsonwebtoken`) + password hashing (`bcryptjs`) |

---

## Table of contents

1. [Features by role](#features-by-role)
2. [Quick start](#quick-start)
3. [Demo accounts](#demo-accounts)
4. [Project structure](#project-structure)
5. [Environment variables](#environment-variables)
6. [npm scripts](#npm-scripts)
7. [Form validation rules](#form-validation-rules)
8. [API overview](#api-overview)
9. [Database design](#database-design)
10. [Engineering decisions](#engineering-decisions)
11. [Troubleshooting](#troubleshooting)
12. [Known limitations](#known-limitations)

---

## Features by role

### System Administrator

- Dashboard with the total number of users, stores and submitted ratings.
- Add new users with name, email, password, address and **role** — so the admin
  can create normal users, store owners and further administrators.
- Add new stores (name, email, address) and assign an owner.
- List every user with Name, Email, Address and Role.
- List every store with Name, Email, Address and Rating.
- Filter any listing by Name, Email, Address and Role.
- Open a single user to see full details; if that user is a Store Owner their
  rating and the stores they own are shown as well.
- Ascending / descending sorting on every key column.
- Log out.

### Normal User

- Sign up from the public registration page (name, email, address, password).
- Log in and log out.
- Update their password after logging in.
- See a list of every registered store with the store name, address, the store's
  **overall rating** and **their own submitted rating**.
- Search stores by **name** and by **address**.
- Submit a rating from 1 to 5 for any store, and modify a rating they already
  submitted.

### Store Owner

- Log in and log out.
- Update their password after logging in.
- Dashboard showing the **average rating** of their store, a breakdown of how
  many of each score they received, and the **list of users who rated them**
  with each user's score and when it was last changed.

---

## Quick start

### Prerequisites

- **Node.js 18 or newer** (`node -v`)
- **PostgreSQL 13 or newer**, either installed locally or run through Docker

### 1. Install dependencies

From the project root:

```bash
npm run install:all
```

(That is shorthand for `npm install` inside `backend/` and inside `frontend/`.)

### 2. Start PostgreSQL

**Option A — Docker (nothing to install besides Docker):**

```bash
docker compose up -d
```

This starts PostgreSQL 16 on `localhost:5432` with user `postgres`, password
`postgres` and a database named `store_rating` — exactly what the default
`DATABASE_URL` expects.

**Option B — PostgreSQL already installed locally:**

Create an empty database once:

```bash
# macOS / Linux
createdb store_rating

# or from inside psql, on any platform
CREATE DATABASE store_rating;
```

### 3. Configure the backend

```bash
cd backend
cp .env.example .env        # Windows: copy .env.example .env
```

Open `backend/.env` and set two values:

- `DATABASE_URL` — must match your PostgreSQL user, password and port.
- `JWT_SECRET` — any long random string. Generate one with:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

### 4. Create the tables and load demo data

```bash
# still inside backend/
npm run db:migrate
npm run db:seed
```

`db:migrate` creates the three tables with their constraints, indexes and
triggers. `db:seed` loads 2 admins, 4 store owners, 8 normal users, 6 stores and
23 ratings so every screen has something to show.

> To wipe and rebuild everything in one step: `npm run db:reset`

### 5. Run the app

Two terminals:

```bash
# terminal 1 — API on http://localhost:5000
cd backend
npm run dev
```

```bash
# terminal 2 — UI on http://localhost:5173
cd frontend
npm run dev
```

Open **http://localhost:5173** and log in with any account below.

> The Vite dev server proxies `/api` to `http://localhost:5000`, so the browser
> only ever talks to one origin and there are no CORS surprises during
> development.

---

## Demo accounts

All created by `npm run db:seed`. The login page also lists them with
click-to-fill buttons.

| Role | Email | Password |
| --- | --- | --- |
| System Administrator | `admin@storerating.com` | `Admin@1234` |
| Normal User | `abhishek@example.com` | `User@1234` |
| Store Owner | `rajesh.owner@example.com` | `Owner@1234` |

Every seeded normal user uses `User@1234` and every seeded store owner uses
`Owner@1234`. `rajesh.owner@example.com` deliberately owns **two** stores, which
is useful for checking that the owner dashboard handles more than one.

---

## Project structure

```
store-rating-app/
├── docker-compose.yml          Optional local PostgreSQL
├── package.json                Convenience scripts for both packages
├── .editorconfig               Shared editor settings
├── .prettierrc.json            Formatting conventions
├── docs/
│   ├── API.md                  Full endpoint reference
│   └── DATABASE.md             Schema, indexes and design rationale
│
├── backend/
│   ├── .env.example
│   ├── .eslintrc.json
│   └── src/
│       ├── server.js           Process entry point, graceful shutdown
│       ├── app.js              Express app and middleware order
│       ├── config/
│       │   ├── env.js          Validated configuration
│       │   └── db.js           pg connection pool + transaction helper
│       ├── constants/roles.js  ADMIN | USER | STORE_OWNER
│       ├── db/
│       │   ├── schema.sql      Tables, constraints, indexes, triggers
│       │   ├── migrate.js      Applies schema.sql
│       │   └── seed.js         Demo data
│       ├── middleware/
│       │   ├── auth.js         authenticate + authorize(...roles)
│       │   ├── errorHandler.js One response shape for every failure
│       │   └── rateLimit.js    Brute-force protection on login/signup
│       ├── utils/
│       │   ├── ApiError.js     Expected failures as a typed error
│       │   ├── asyncHandler.js Forwards async rejections to Express
│       │   ├── validation.js   The assignment's form rules
│       │   ├── queryOptions.js Sort whitelist, pagination, safe ILIKE
│       │   ├── jwt.js
│       │   ├── password.js
│       │   └── serialize.js    Row -> JSON, strips password_hash
│       ├── controllers/        auth | admin | store | rating | owner
│       └── routes/             One router per audience
│
└── frontend/
    ├── vite.config.js          Dev server + /api proxy
    ├── .eslintrc.json
    └── src/
        ├── main.jsx            Router + AuthProvider + App
        ├── App.jsx             Route tree with role gating
        ├── styles.css          Whole design system
        ├── api/
        │   ├── client.js       fetch wrapper, token handling, errors
        │   └── endpoints.js    One function per endpoint
        ├── context/AuthContext.jsx
        ├── hooks/useDebouncedValue.js
        ├── components/         Layout, RouteGuards, DataTable, Pagination,
        │                       StarRating, Modal, FormField, Alert, StatCard...
        ├── utils/              constants, validation (mirrored), format
        └── pages/
            ├── LoginPage.jsx, SignupPage.jsx, UpdatePasswordPage.jsx
            ├── admin/          Dashboard, Users, UserDetail, AddUser,
            │                   Stores, AddStore
            ├── user/           UserStoresPage (browse + rate)
            └── owner/          OwnerDashboardPage
```

---

## Environment variables

`backend/.env` (see `backend/.env.example` for the annotated version):

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `5000` | Port the API listens on |
| `NODE_ENV` | `development` | Enables request logging and stack traces |
| `DATABASE_URL` | `postgresql://postgres:postgres@localhost:5432/store_rating` | PostgreSQL connection string |
| `DATABASE_SSL` | `false` | Set `true` for hosted databases that require SSL |
| `DATABASE_SSL_REJECT_UNAUTHORIZED` | `true` | Verify the database's TLS certificate; only disable for self-signed certs |
| `TRUST_PROXY` | `false` | Set `true` only when running behind a reverse proxy, so `X-Forwarded-For` cannot be spoofed past the rate limiter |
| `JWT_SECRET` | *(required)* | Signing key; must be 32+ characters |
| `JWT_EXPIRES_IN` | `1d` | Token lifetime |
| `CLIENT_ORIGIN` | `http://localhost:5173` | Origin allowed by CORS |
| `BCRYPT_SALT_ROUNDS` | `10` | bcrypt cost factor |
| `SEED_ADMIN_NAME` / `SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` | see example file | First admin created by the seed script |

`frontend/.env` is optional — only needed if the API is not reachable at `/api`
(see `frontend/.env.example`).

---

## npm scripts

Run from the project root:

| Script | What it does |
| --- | --- |
| `npm run install:all` | Installs backend and frontend dependencies |
| `npm run db:migrate` | Creates the schema |
| `npm run db:seed` | Loads demo data (truncates first) |
| `npm run db:reset` | Drops, recreates and re-seeds |
| `npm run dev:backend` | Starts the API with nodemon |
| `npm run dev:frontend` | Starts the Vite dev server |
| `npm run build:frontend` | Production build into `frontend/dist` |
| `npm run lint` | Runs ESLint over both packages |

Inside `backend/` there is additionally `npm start` (production, no nodemon) and
`npm run db:migrate:fresh`.

---

## Form validation rules

Exactly as specified in the assignment, and enforced in **three** places:
instantly in the browser for feedback, authoritatively in the API, and as
`CHECK` constraints in PostgreSQL so bad data cannot be written by any route.

| Field | Rule |
| --- | --- |
| Name | Minimum 20 characters, maximum 60 |
| Address | Maximum 400 characters |
| Password | 8–16 characters, at least one uppercase letter and one special character |
| Email | Standard email format, stored and compared case-insensitively |
| Rating | Integer from 1 to 5 |

**Interpretation note:** the 20–60 character name rule is applied to store names
as well as user names, because the store form has the same Name field and the
specification lists the rule once for "Name". That is why the seeded store names
are written out in full (for example *Shree Ganesh Super Market*).

---

## API overview

Base URL `http://localhost:5000/api`. Every response uses the same envelope:

```json
{ "success": true,  "data": { }, "message": "optional" }
{ "success": false, "message": "...", "fieldErrors": { "email": "..." } }
```

| Method | Endpoint | Role | Purpose |
| --- | --- | --- | --- |
| `POST` | `/auth/signup` | public | Register a normal user |
| `POST` | `/auth/login` | public | Log in (all three roles) |
| `GET` | `/auth/me` | any | Current user, used to restore a session |
| `PUT` | `/auth/password` | any | Update password |
| `GET` | `/admin/dashboard` | ADMIN | Totals for users / stores / ratings |
| `GET` | `/admin/users` | ADMIN | Filter + sort + paginate users |
| `POST` | `/admin/users` | ADMIN | Create a user with any role |
| `GET` | `/admin/users/:id` | ADMIN | User detail (+ rating if store owner) |
| `GET` | `/admin/stores` | ADMIN | Filter + sort + paginate stores |
| `POST` | `/admin/stores` | ADMIN | Create a store and assign an owner |
| `GET` | `/admin/owner-options` | ADMIN | Candidates for the owner dropdown |
| `GET` | `/stores` | USER | Stores with overall rating + your rating |
| `GET` | `/stores/:id` | USER | One store, same computed values |
| `POST` | `/ratings` | USER | Submit a rating |
| `PUT` | `/ratings/:storeId` | USER | Modify your rating |
| `GET` | `/owner/dashboard` | STORE_OWNER | Average, breakdown and rater list |
| `GET` | `/health` | public | Liveness check |

Full request/response examples, query parameters and status codes:
**[docs/API.md](docs/API.md)**.

---

## Database design

```
users                         stores                        ratings
-----                         ------                        -------
id            PK              id           PK               id          PK
name          20-60 chars     name         20-60 chars      user_id     FK -> users
email         unique (lower)  email        unique (lower)   store_id    FK -> stores
password_hash                 address      <= 400 chars     rating      1..5
address       <= 400 chars     owner_id     FK -> users      created_at
role          ADMIN|USER|      created_at                    updated_at
              STORE_OWNER     updated_at
created_at                                                  UNIQUE (user_id, store_id)
updated_at
```

Two decisions are worth calling out:

- **`UNIQUE (user_id, store_id)` on `ratings`** is the rule "one user rates a
  store once" expressed in the database. Enforcing it there rather than in
  application code means two simultaneous requests cannot both slip a duplicate
  through, and it is what lets the listing query fetch "your rating" with a
  single `MAX(CASE WHEN ...)` instead of a sub-query.
- **Average ratings are computed, not stored.** There is no `average_rating`
  column on `stores`; every average comes from `AVG(rating)` at query time, so
  there is exactly one source of truth and no cache to invalidate.

Rationale, index list and how this would scale: **[docs/DATABASE.md](docs/DATABASE.md)**.

---

## Engineering decisions

**Single login, role-based routing.** There is one `POST /auth/login` for all
three roles. The JWT carries the user id and role, but `authenticate` re-reads
the user row from the database on every request — so if an admin changes
someone's role, it takes effect on the next request instead of whenever the old
token happens to expire.

**Authorisation lives in the API, not the UI.** `authorize(...roles)` guards
every protected router. The React guards in `components/RouteGuards.jsx` exist
only to give the right experience; hiding a button stops nobody from calling the
endpoint with `curl`, so the server never trusts the client.

**No SQL injection surface.** All values go through `$1`-style parameters. Column
names and `ASC`/`DESC` cannot be parameterised, so sorting uses a **whitelist**:
the client sends a key such as `sortBy=rating`, and the server looks that key up
in a map of allowed columns and uses its own SQL string. Search terms are also
escaped before being placed into an `ILIKE` pattern, so searching for `100%` does
not match everything.

**Server-side filtering, sorting and pagination.** Sorting a page of 10 rows in
the browser would produce the wrong order, because row 11 may belong at the top.
The server does all three and returns the page plus metadata.

**One error shape.** Expected failures are thrown as `ApiError`; a single error
handler turns them into `4xx` responses with field-level messages, maps
PostgreSQL error codes (`23505`, `23514`, `23503`, …) to readable text, and
replies to anything unexpected with a generic 500 while logging the real stack
trace server-side.

**Passwords.** Hashed with bcrypt (cost 10) and never selected into any response
— `serialize.js` builds the user payload, so `password_hash` has one place it
could leak from and does not. Login returns a deliberately vague "Incorrect
email or password" so the endpoint cannot be used to discover which addresses are
registered, and both `/auth/login` and `/auth/signup` are rate limited.

**Transactions where they matter.** Creating a store can also promote the chosen
owner from `USER` to `STORE_OWNER`; both statements run in one transaction with
`SELECT ... FOR UPDATE` on the owner row, so the store and the role change
succeed or fail together.

**Accessibility.** Tables expose `aria-sort`, form fields are wired to their
labels, hints and errors with `htmlFor` / `aria-describedby` / `aria-invalid`,
the rating picker is made of real buttons so it works from the keyboard, and the
modal closes on `Escape` and restores focus to whatever opened it.

---

## Troubleshooting

**`Cannot reach the database` / `ECONNREFUSED`**
PostgreSQL is not running or `DATABASE_URL` is wrong. With Docker, check
`docker compose ps`. Locally, confirm the port and that the database exists.

**`Database tables are missing`**
Run `npm run db:migrate` inside `backend/`.

**`password authentication failed for user "postgres"`**
Update the user and password inside `DATABASE_URL` in `backend/.env` to match
your local PostgreSQL installation.

**`Port 5432 is already allocated` when starting Docker**
A local PostgreSQL is already using the port. Either stop it, or change the host
port in `docker-compose.yml` (for example `"5433:5432"`) and update
`DATABASE_URL` to use `5433`.

**`Cannot reach the server` in the browser**
The frontend is up but the API is not. Start it with `npm run dev` in `backend/`
and confirm `http://localhost:5000/api/health` responds.

**Seed script fails with a constraint violation**
Every name must be at least 20 characters. If you edited the seed data, check
the names you added.

---

## Known limitations

Honest list of what was consciously left out, and what the next step would be:

- **No automated tests.** The highest-value additions would be integration tests
  over the rating endpoints (duplicate submission, cross-user tampering) and the
  sort whitelist.
- **No refresh tokens.** A JWT cannot be revoked before it expires; logging out
  discards it client-side. A short-lived access token plus a revocable refresh
  token would be the production answer.
- **Rate limiting is in-process.** The counter lives in a `Map`, so it resets on
  restart and is per-instance. Behind more than one instance it belongs in Redis
  or at the proxy.
- **Averages are recomputed on every read.** Correct and fast at this data size.
  Past a few hundred thousand ratings the answer is a materialised view or
  a maintained `rating_sum` / `rating_count` pair on `stores`, refreshed by a
  trigger.
- **Validation rules exist in three places** (client, API, database). That is
  intentional defence in depth, but it means a rule change needs three edits —
  a shared validation package would remove the duplication.
- **Admins cannot edit or delete** users and stores; the assignment only asked
  for creation and listing, so the destructive operations were left out rather
  than half-built.
