# ExpenseFlow — Expense Reimbursement Management System

 **Express (Node.js) + PostgreSQL + React (Vite)** with **JWT auth**, **role-based access**, **multi-step approval workflows** (percentage / specific / hybrid + dynamic manager step), **Tesseract.js OCR** on receipts, **Frankfurter** FX conversion to company base currency, **Socket.io** real-time notifications, **dashboard analytics** (Recharts), **PDF/CSV export**, and **smart spend insights**.

## Architecture

- **MVC-style backend**: `routes` → `controllers` → `models` (SQL via `pg`) + `services` (OCR, workflow engine, currency, exports, insights).
- **Frontend**: React 18, React Router, Axios, Recharts, Socket.io client.
- **Database**: PostgreSQL (parameterized queries — SQL injection safe).

### Folder structure

```
ODOOHACKS/
├── database/
│   └── schema.sql              # Full DDL + indexes
├── server/
│   ├── package.json
│   ├── .env.example
│   ├── scripts/init-db.js      # Optional: apply schema via Node
│   └── src/
│       ├── server.js           # HTTP + Socket.io
│       ├── app.js
│       ├── config/             # env, db pool
│       ├── middleware/         # auth, upload, errors
│       ├── models/             # data access
│       ├── controllers/
│       ├── services/           # domain logic
│       ├── routes/
│       └── utils/socketHub.js
└── client/
    ├── package.json
    ├── vite.config.js
    └── src/
        ├── api/
        ├── context/
        ├── hooks/
        ├── components/
        └── pages/
```

## Prerequisites

- **Node.js 18+**
- **PostgreSQL 14+** (local or Docker)

## Step-by-step setup

### 1. Create database

```bash
# Using psql (adjust user/db names)
createdb expense_reimbursement
psql -U postgres -d expense_reimbursement -f database/schema.sql
```

Or use the optional script (requires `DATABASE_URL`):

```bash
cd server
copy .env.example .env
# Edit .env: set DATABASE_URL and JWT_SECRET
npm run db:init
```

### 2. Backend

```bash
cd server
npm install
copy .env.example .env
```

Edit **`server/.env`**:

| Variable | Example |
|----------|---------|
| `DATABASE_URL` | `postgresql://postgres:postgres@localhost:5432/expense_reimbursement` |
| `JWT_SECRET` | Long random string |
| `CLIENT_URL` | `http://localhost:5173` |
| `PORT` | `5000` |

```bash
npm run dev
```

API base: `http://localhost:5000/api`  
Health: `GET /api/health`  
Uploads served at `/uploads/...`

### 3. Frontend

```bash
cd client
npm install
npm run dev
```

Open `http://localhost:5173`. Vite proxies `/api`, `/uploads`, and `/socket.io` to the API in development.

**Production / split hosts**: set `client/.env`:

```env
VITE_API_URL=http://localhost:5000
VITE_WS_URL=http://localhost:5000
```

### 4. First user

Use **Sign up** — this creates:

1. A **company** (with chosen **base currency**).
2. An **admin** user.
3. A **default workflow**: step 1 = submitter’s **manager** (dynamic); step 2 = **60% quorum** among users with the **admin** role in the pool (see `authService.seedDefaultWorkflow`).

Then create **employees** and **managers** under **Users**, set **manager** IDs for hierarchy.

## API overview (REST)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/auth/signup` | — | Company + admin + default workflow |
| POST | `/api/auth/login` | — | JWT |
| GET | `/api/auth/me` | JWT | Profile + company |
| GET/POST/PATCH | `/api/users` | Admin | List / create / update users |
| GET/POST | `/api/workflows` | Admin | List / create workflow + steps |
| POST | `/api/expenses` | JWT | Multipart: expense fields + optional `receipt` |
| POST | `/api/expenses/ocr-preview` | JWT | OCR only |
| GET | `/api/expenses/me` | JWT | Own expenses |
| GET | `/api/expenses/company` | Admin, Manager | All company expenses |
| GET | `/api/expenses/pending-for-me` | JWT | Approval queue |
| POST | `/api/expenses/:id/submit` | JWT | Draft → pending |
| POST | `/api/expenses/:id/decision` | JWT | `{ action, comment }` |
| GET | `/api/dashboard/summary` | JWT | Stats + charts data |
| GET | `/api/notifications` | JWT | In-app notifications |
| GET | `/api/exports/expenses.csv` | Admin, Manager | CSV |
| GET | `/api/exports/expenses.pdf` | Admin, Manager | PDF |

Errors return JSON: `{ success: false, error: { code, message } }` with appropriate HTTP status.

## Workflow rules (JSON `rule_config`)

- **specific** + `{ "dynamic": "manager" }` — approver = expense submitter’s `manager_id`.
- **specific** + `{ "user_id": 5 }` or `{ "role_id": 2 }` — designated user or any user with that role.
- **percentage** + `{ "quorum": 0.6, "pool_role_ids": [2] }` — need ceil(quorum × pool size) approvals from users in those roles (reject from pool fails the step).
- **hybrid** + `{ "quorum": 0.5, "must_include_user_ids": [3], "pool_role_ids": [2] }` — must-include users must approve, plus quorum on pool.

## Security notes

- Passwords: **bcrypt**.
- Input validation: **express-validator** on controllers.
- SQL: **parameterized** queries only.
- Files: **multer** size/type limits; receipts stored under `server/uploads`.

## MySQL variant

The schema and queries target **PostgreSQL** (`JSONB`, `date_trunc`, etc.). For **MySQL**, port `database/schema.sql` (JSON columns, syntax) and swap `pg` for `mysql2/promise` in `config/db.js` and models.

## Production build (single origin)

The API can serve the built React app so one URL hosts UI + REST + WebSockets.

1. Build the client: `cd client && npm run build` (creates `client/dist`).
2. Set `NODE_ENV=production`, `DATABASE_URL`, `JWT_SECRET`, and **`CLIENT_URL`** to the public URL users open (e.g. `http://localhost:8080` or `https://your-app.onrender.com`).
3. From `server`: `npm run start:prod` (runs `scripts/docker-entrypoint.mjs`: applies `database/schema.sql` idempotently, then starts the server).

CORS and Socket.io use **`CLIENT_URL`** (comma-separated for multiple origins). Behind a reverse proxy (Render, Railway), set **`TRUST_PROXY=1`**.

## Docker (recommended for a full stack on one machine)

Requires [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/macOS) or Docker Engine (Linux).

From the repo root:

```bash
docker compose up --build
```

- **App**: [http://localhost:8080](http://localhost:8080) — UI, `/api`, `/uploads`, Socket.io on the same origin.
- **PostgreSQL**: port `5432` (user `expense`, password `expense`, database `expense_reimbursement`).
- Uploads persist in the `uploads` Docker volume.

Override secrets: `set JWT_SECRET=...` (PowerShell) or `export JWT_SECRET=...` (bash) before `docker compose up`.

## Hosting on Render (free tier friendly)

1. Push this project to GitHub.
2. In [Render](https://render.com), create a **PostgreSQL** instance; copy its **Internal Database URL**.
3. Create a **Web Service** → Connect the repo → **Docker** → Dockerfile path `./Dockerfile`, context `.`.
4. Set environment variables:

   | Key | Value |
   |-----|--------|
   | `DATABASE_URL` | PostgreSQL URL from step 2 |
   | `JWT_SECRET` | Long random string |
   | `NODE_ENV` | `production` |
   | `CLIENT_URL` | Your web service URL, e.g. `https://<name>.onrender.com` |
   | `TRUST_PROXY` | `1` |
   | `PORT` | `5000` (Render usually injects `PORT`; keep default if unset) |

5. Deploy. Open the public URL → **Sign up** to create the first company and admin.

**Note:** Free web services spin down when idle; first request may take ~30–60s.

## Hosting on Railway / Fly.io / VPS

Same as Render: run the **Dockerfile** (or `node` + `npm run start:prod` with `client/dist` present), provide PostgreSQL `DATABASE_URL`, set `CLIENT_URL` to the HTTPS origin users use, and `TRUST_PROXY=1` if behind a proxy.

## License

Hackathon / educational use.
