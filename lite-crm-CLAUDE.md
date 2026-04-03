# CLAUDE.md

This file gives you, Claude Code, everything you need to help me build and run my CRM app. Read this before doing anything else. Follow these instructions exactly — they are my source of truth for how this project works.

---

## What I Am Building

I am building a lightweight desktop CRM called **Lite CRM** as an Electron application. It is for two people on my small sales team. We share a single cloud database so our contacts, accounts, deals, and activity are all in sync.

**Core features — nothing more:**
- Accounts (companies I sell to)
- Contacts (people at those companies)
- Deals (sales opportunities)
- Activity (emails, calls, meetings, notes logged against contacts and deals)
- Gmail Sync (pulls sent/received emails into activity automatically)

I am not an engineer. When I need to install something or run a command, give me step-by-step instructions I can follow exactly.

---

## My Machine Setup

Before starting any code work, ask me if I have completed the following. If I have not, walk me through each item:

### 1. Install Node.js
Download the LTS version from https://nodejs.org — choose the macOS or Windows installer and run it. Verify by opening Terminal (Mac) or Command Prompt (Windows) and running:
```
node --version
npm --version
```
Both should print a version number.

### 2. Install VS Code (optional but recommended)
Download from https://code.visualstudio.com. This gives me a place to view files and a built-in terminal.

### 3. Create a `.env` file
Once you scaffold the project, I will create a `.env` file in the project root with my secrets. You will tell me what values to put in it. Never commit this file to git.

---

## Tech Stack

Use exactly this stack. Do not substitute alternatives.

| Layer | Choice |
|---|---|
| Desktop shell | Electron 29 |
| Frontend | React 18 + TailwindCSS 3 + Vite 5 |
| Backend | Express (embedded in Electron main process) |
| Database queries | Knex.js with pg driver |
| Database | PostgreSQL (Supabase or RDS — see below) |
| Packaging | Electron Forge |

---

## Architecture

```
Electron main process
  └── starts Express on port 3850
        ├── /api/accounts
        ├── /api/contacts
        ├── /api/deals
        ├── /api/activities
        └── /api/gmail
  └── opens BrowserWindow → Vite dev server (dev) or dist/index.html (prod)

React frontend (src/)
  └── calls Express via fetch('/api/...')
  └── Pages: Accounts, Contacts, Deals, Activity, Settings
```

### Server entry point
`server/index.js` exports an async `createServer()` function that:
1. Creates a Knex instance from `DATABASE_URL`
2. Attaches `req.db` (the Knex instance) to every request via middleware
3. Mounts all route files
4. Returns the HTTP server

### Electron main entry point
`electron/main.js`:
1. Calls `createServer()` and starts listening on port 3850
2. Creates a `BrowserWindow`
3. In dev: loads `http://localhost:5173` (Vite)
4. In production: loads `dist/index.html`

### Frontend entry point
`src/main.jsx` → `src/App.jsx` (routing + page state)

---

## Database

I have not decided between Supabase and RDS yet. The app must work with either. The only config point is `DATABASE_URL` in my `.env`.

### Supabase (preferred — easier to set up)
1. Go to https://supabase.com and create a free account
2. Create a new project and wait for it to provision
3. Go to Project Settings → Database → Connection string → URI
4. Copy the URI (starts with `postgresql://`) and put it in `.env` as `DATABASE_URL`
5. No SSL cert file needed — Supabase handles it

### RDS (alternative)
1. `DATABASE_URL` is the connection string from the RDS console
2. If SSL is required, I will have a `global-bundle.pem` file — tell me where to put it

### Knex config (`server/db.js`)
```js
const knex = require('knex')({
  client: 'pg',
  connection: process.env.DATABASE_URL,
  ssl: process.env.DATABASE_URL?.includes('supabase') ? { rejectUnauthorized: false } : undefined,
});
module.exports = knex;
```
Attach to every request: `app.use((req, _res, next) => { req.db = knex; next(); });`

---

## Database Schema

Create these tables via SQL migrations in `db/migrations/`. Run them in order using `npm run db:migrate`. Each file is plain SQL. The migration runner reads them alphabetically and executes via Knex raw.

### `001_schema.sql`
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS accounts (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  domain TEXT,
  industry TEXT,
  size TEXT,
  status TEXT DEFAULT 'prospect', -- prospect, customer, churned
  owner TEXT,                      -- rep email
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS contacts (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  first_name TEXT NOT NULL,
  last_name TEXT,
  email TEXT,
  phone TEXT,
  title TEXT,
  account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
  owner TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS deals (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  value NUMERIC,
  stage TEXT DEFAULT 'Targeting',
  -- stages: Targeting, Meeting Booked, Discovery, Demo, Evaluation, Negotiation, Closed Won, Closed Lost
  account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
  contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,
  owner TEXT,
  expected_close DATE,
  next_steps TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS activities (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  type TEXT NOT NULL,              -- email, call, meeting, note, linkedin
  direction TEXT,                  -- inbound, outbound
  subject TEXT,
  description TEXT,
  date TIMESTAMPTZ DEFAULT NOW(),
  account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
  contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,
  deal_id UUID REFERENCES deals(id) ON DELETE SET NULL,
  owner TEXT,
  gmail_message_id TEXT UNIQUE,
  gmail_thread_id TEXT,
  from_email TEXT,
  to_emails TEXT[],
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_activities_account ON activities(account_id);
CREATE INDEX IF NOT EXISTS idx_activities_contact ON activities(contact_id);
CREATE INDEX IF NOT EXISTS idx_activities_deal ON activities(deal_id);
CREATE INDEX IF NOT EXISTS idx_activities_date ON activities(date DESC);
CREATE INDEX IF NOT EXISTS idx_contacts_account ON contacts(account_id);
CREATE INDEX IF NOT EXISTS idx_deals_account ON deals(account_id);
```

### `002_gmail.sql`
```sql
CREATE TABLE IF NOT EXISTS oauth_tokens (
  user_email TEXT PRIMARY KEY,
  tokens JSONB NOT NULL,
  display_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS gmail_sync_state (
  user_email TEXT PRIMARY KEY,
  history_id TEXT,
  last_sync_at TIMESTAMPTZ,
  status TEXT DEFAULT 'idle',
  error_message TEXT
);

CREATE TABLE IF NOT EXISTS personal_domains (
  domain TEXT PRIMARY KEY
);

-- Block consumer email domains to avoid syncing personal emails
INSERT INTO personal_domains (domain) VALUES
  ('gmail.com'), ('yahoo.com'), ('hotmail.com'), ('outlook.com'),
  ('icloud.com'), ('aol.com'), ('protonmail.com'), ('me.com')
ON CONFLICT DO NOTHING;
```

---

## Gmail Sync

### Overview
Each user on a machine authenticates their Gmail once. The app then polls Gmail every 60 seconds for new emails and logs them as activities, matching sender/recipient domains to accounts.

### What I need from Google Cloud Console
Walk me through these steps when I ask to set up Gmail:
1. Go to https://console.cloud.google.com
2. Create a new project (name it anything)
3. Enable the **Gmail API** (APIs & Services → Enable APIs → search Gmail)
4. Go to APIs & Services → OAuth consent screen → External → fill in app name and my email
5. Add scopes: `gmail.readonly` and `userinfo.email`
6. Add my email as a test user
7. Go to Credentials → Create Credentials → OAuth client ID → Desktop app
8. Download the JSON — copy `client_id` and `client_secret` into `.env`

### `.env` variables for Gmail
```
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://127.0.0.1:3850/api/gmail/oauth/callback
```

### Sync logic (`server/services/gmail-sync.js`)
- On first auth: run initial sync — query Gmail for emails involving my company domain (last 2 years), not all mail
- On subsequent syncs: use `history.list()` with stored `history_id` for incremental updates
- For each email: extract sender, recipients, subject, date, body snippet
- Domain match: strip `@` from emails, look up in `accounts.domain` — if match, link `account_id`
- Also try to match `contacts.email` directly
- Skip emails where both sender and recipient are personal domains (blocklist in `personal_domains` table)
- Store result as an activity with `type='email'`, `gmail_message_id` as dedup key

### OAuth routes (`server/routes/gmail.js`)
- `GET /api/gmail/auth-url` — returns Google OAuth URL
- `GET /api/gmail/oauth/callback` — exchanges code for tokens, stores in `oauth_tokens`, starts sync
- `GET /api/gmail/status` — returns sync status and authenticated user
- `POST /api/gmail/disconnect` — removes tokens and sync state

---

## Key Features Per Page

### Accounts (`/accounts`)
- Table: Name, Domain, Industry, Status, Owner, Last Activity date
- Click row → Account detail: info fields + contact list + deal list + activity timeline
- Add/edit account inline or via modal
- Filter by Status, Owner

### Contacts (`/contacts`)
- Table: Name, Title, Account, Email, Owner, Last Activity
- Click row → Contact detail: info fields + linked deals + activity timeline
- Add/edit contact
- Filter by Account, Owner

### Deals (`/pipeline`)
- Kanban board grouped by stage (or list view toggle)
- Card: Deal name, Account, Value, Expected Close, Owner
- Click card → Deal detail: all fields + activity timeline
- Drag card between stages (updates `stage` field)
- Filter by Owner, Stage

### Activity (`/activity`)
- Chronological feed of all activity across accounts/contacts/deals
- Shows: type icon, subject, account name, contact name, date, direction
- Manual log form: type, subject, description, link to account/contact/deal
- Gmail-synced entries are read-only (identified by `gmail_message_id` being set)
- Filter by type, owner, date range

### Settings (`/settings`)
- Gmail: connect/disconnect button, sync status, last synced timestamp
- My profile: display name, email (read from oauth_tokens)

---

## Multi-User Setup (2 People)

Both users install the Electron app on their own machines. Both connect to the same database using the same `DATABASE_URL`. Each user connects their own Gmail account in Settings. All data (accounts, contacts, deals, activities) is shared in real-time through the database.

To give a second user access:
1. Send them the packaged app (DMG or installer)
2. Send them the `.env` file with `DATABASE_URL` and Google OAuth credentials
3. They install, open Settings, and connect their Gmail

The `owner` field on accounts, contacts, deals, and activities stores the rep's email. Use this for filtering ("My Accounts", "All Accounts").

---

## Build and Distribution

### Development
```
npm run dev
```
Runs Vite dev server (port 5173) + starts Electron pointing at it.

### Production build
```
npm run package
```
Runs Electron Forge — outputs a DMG (Mac) or installer (Windows) to `out/`.

### `package.json` scripts
```json
{
  "scripts": {
    "dev": "concurrently \"vite\" \"electron .\"",
    "build": "vite build",
    "package": "electron-forge package",
    "make": "electron-forge make",
    "db:migrate": "node scripts/run-migrations.mjs"
  }
}
```

### Migration runner (`scripts/run-migrations.mjs`)
Read all `.sql` files from `db/migrations/` alphabetically, execute each via Knex raw in sequence. Log success/failure per file. Do not use transactions (keep it simple).

### Packaging with Electron Forge
- `forge.config.js`: use `@electron-forge/maker-dmg` (Mac) and `@electron-forge/maker-squirrel` (Windows)
- Bundle `.env` as an extraResource so it ships with the packaged app
- At runtime, load `.env` from `process.resourcesPath` first, then fall back to project root

---

## UI Conventions

- No emojis anywhere in the UI
- Use first names for display (extract from `display_name` or `owner` email)
- Dense, compact layouts — minimize whitespace
- Consistent table formatting across all pages
- `owner` fields display as first name, not email
- Derived fields (e.g., "Last Activity" computed from activities table) are read-only — do not make them editable
- Dark/light mode toggle in Settings is optional but nice to have

---

## Env File Template

When the project is scaffolded, create a `.env.example` file with this content. I will copy it to `.env` and fill in my values:

```
# Database (Supabase or RDS connection string)
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Gmail OAuth (from Google Cloud Console)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=http://127.0.0.1:3850/api/gmail/oauth/callback
```

---

## Gotchas

- **`gmail_message_id` is the dedup key** — always use `INSERT ... ON CONFLICT (gmail_message_id) DO NOTHING` when inserting synced emails. Never upsert them.
- **Personal domain blocklist** — before matching an email to an account, check both sender and all recipients against `personal_domains`. Skip the email entirely if both sides are personal domains.
- **Domain matching is not exact** — strip subdomains when matching. `mail.boeing.com` should match account with domain `boeing.com`. Split on `.`, take last two parts.
- **Knex SSL** — Supabase requires `ssl: { rejectUnauthorized: false }`. RDS may require a cert file. Handle both via `DATABASE_URL` inspection or an `SSL_MODE` env var.
- **Electron + Vite in dev** — Electron must wait for Vite to be ready before loading the URL. Add a `wait-on` check in the dev script: `wait-on http://localhost:5173 && electron .`
- **Port 3850** — Express always runs on 3850 inside the Electron process. The React frontend fetches from `/api/...` (relative) in production (same-origin), and from `http://127.0.0.1:3850/api/...` in dev.
- **Token storage** — OAuth tokens go in the `oauth_tokens` table (not a file). This ensures both users can auth independently and tokens survive app restarts.
- **History ID must be saved after every sync** — if it is lost, the next sync will re-import everything from scratch (expensive and slow).
- **No ORM** — all DB queries use Knex's query builder. No Prisma, Sequelize, or TypeORM. Keep queries simple and readable.
- **No pagination yet** — API routes return all rows. This is fine for a small team. Add pagination later if needed.
