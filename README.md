# Pinora — Visual Discovery & Image Organisation Platform

**CWI Task 52 — Pinterest clone (functional, web-based).**

Pinora is a fully functional, original visual-discovery platform: search and browse
inspiration, upload images, create pins, organise them into boards, like, comment and
manage a profile. Every piece of application data lives in a **Turso (libSQL)** database —
nothing important is kept in `localStorage`, hardcoded arrays or in-memory objects.

> **No fake functionality.** Every button, form and navigation link performs a real
> database operation. A `200 OK` means the row was committed. If a write fails the UI
> shows an error instead of pretending it succeeded.

---

## Table of contents

- [Project overview](#project-overview)
- [Feature checklist](#feature-checklist)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Turso setup](#turso-setup)
- [Environment variables](#environment-variables)
- [Database initialisation](#database-initialisation)
- [Migrations](#migrations)
- [Seed data](#seed-data)
- [Database schema](#database-schema)
- [Database relationships](#database-relationships)
- [Authentication](#authentication)
- [Pin system](#pin-system)
- [Board system](#board-system)
- [Search](#search)
- [Filtering](#filtering)
- [Image storage approach](#image-storage-approach)
- [API reference](#api-reference)
- [Local development](#local-development)
- [Build instructions](#build-instructions)
- [Deployment instructions](#deployment-instructions)
- [Verification & testing](#verification--testing)
- [Security notes](#security-notes)
- [Accessibility & responsiveness](#accessibility--responsiveness)
- [Troubleshooting](#troubleshooting)
- [Licence](#licence)

---

## Project overview

Pinora is a single-page application served by a small Node.js API. The API owns the
database connection, so **Turso credentials never reach the browser**.

- **Frontend** — dependency-free ES modules (`app/public`): history-based router,
  masonry feed, modals, toasts, optimistic-free UI that always re-reads state from the API.
- **Backend** — Fastify 5 REST API (`server/`) with server-side sessions, bcrypt password
  hashing, ownership checks on every mutation and structured error handling.
- **Database** — Turso / libSQL accessed through `@libsql/client`, with ordered SQL
  migrations, seed data, foreign keys, constraints, indexes and `updated_at` triggers.

The visual language (brand, palette, layout, motion) is original work; no Pinterest
source code, assets or proprietary implementation is used.

---

## Feature checklist

| Requirement | Status | Where |
| --- | --- | --- |
| Visual discovery homepage (trending / popular / fresh) | ✅ | `app/public/js/views/home.js` |
| Explore page with trending, popular, recent, categories, creators | ✅ | `app/public/js/views/explore.js` |
| Responsive masonry grid with natural aspect ratios | ✅ | `app/public/css/styles.css`, `components.js` |
| Search by title, description, category, creator | ✅ | `server/queries.js` (`pinListing`) |
| Category filtering + combined search & category | ✅ | `views/search.js`, `GET /api/pins` |
| Real image upload from the device | ✅ | `server/upload.js`, `POST /api/uploads` |
| Create pin (image, title, description, category, destination, board) | ✅ | `views/pin.js` |
| Edit pin | ✅ | `views/pin.js`, `PATCH /api/pins/:id` |
| Delete pin (soft delete, boards preserved) | ✅ | `DELETE /api/pins/:id` |
| Pin detail page (image, meta, creator, stats, comments, related) | ✅ | `views/pin.js` |
| Save pin to a board (with duplicate protection) | ✅ | `POST /api/boards/:id/pins` |
| Remove pin from a board | ✅ | `DELETE /api/boards/:id/pins/:pinId` |
| Create / edit / delete boards, privacy toggle | ✅ | `views/boards.js`, `actions.js` |
| My boards page + saved pins tab | ✅ | `views/boards.js` |
| Board detail page (cover, title, description, pins) | ✅ | `views/boards.js` |
| Likes (like / unlike, no duplicates) | ✅ | `POST|DELETE /api/pins/:id/like` |
| Comments (add, list, delete own) | ✅ | `views/pin.js`, `actions.js` |
| Related pins (same category, then recent fallback) | ✅ | `server/queries.js#getRelatedPins` |
| Share (Web Share API with clipboard fallback) | ✅ | `actions.js#share-pin` |
| Signup / login / logout / session persistence | ✅ | `server/auth.js`, `views/auth.js` |
| Profile view + profile management (name, username, bio, avatar) | ✅ | `views/profile.js` |
| Follow / unfollow creators | ✅ | `POST|DELETE /api/users/:id/follow` |
| Notifications (real engagement on your pins) | ✅ | `GET /api/notifications` |
| Loading states (skeletons, button spinners) | ✅ | `ui.js#skeletonGrid`, `skeletonProfile`, `skeletonBoardHeader`, `setBusy` |
| Error states (friendly, no SQL/stack traces) | ✅ | `server/routes.js#setErrorHandler` |
| Empty states with CTAs | ✅ | `views/*`, `components.js#masonryHtml` |
| Toast notifications | ✅ | `ui.js#toast` |
| Confirmation modals for destructive actions | ✅ | `ui.js#confirmDialog` |
| 404 page | ✅ | `views/profile.js#notFoundView` |
| Mobile drawer navigation | ✅ | `components.js#drawerHtml`, `actions.js` |
| Dark mode, reduced-motion support | ✅ | `app/public/css/styles.css` |
| Migrations, seed data, schema snapshot | ✅ | `db/` |

---

## Tech stack

| Layer | Choice | Why |
| --- | --- | --- |
| Runtime | Node.js ≥ 20 (ESM) | Native `fetch`, top-level `await`, `crypto.randomUUID` |
| Server | [Fastify 5](https://fastify.dev) | Fast, schema-free routing that keeps error text under our control |
| Cookie/session | `@fastify/cookie` + DB-backed sessions | httpOnly cookie, revocable server-side |
| Uploads | `@fastify/multipart` | Streamed multipart handling with size limits |
| Static files | `@fastify/static` | Serves the SPA and `/uploads/*` |
| Database | **Turso / libSQL** via `@libsql/client` | Assignment requirement; identical client for remote and local |
| Passwords | `bcryptjs` | Never store plain text |
| Frontend | Vanilla ES modules + CSS custom properties | No build step, no framework lock-in, instant deploys |
| Typography | **Plus Jakarta Sans** (UI) + **Clash Display** (headings/brand) | Self-hosted from `app/public/fonts/`, no third-party font CDN |

---

## Project structure

```
.
├── app/public/                 # Frontend (served statically, no build step)
│   ├── index.html              # App shell
│   ├── favicon.svg
│   ├── css/styles.css          # Design system, masonry, responsive rules, motion
│   └── js/
│       ├── main.js             # Boot: routes, global image fallback
│       ├── router.js           # History API router
│       ├── api.js              # fetch wrapper + endpoint map
│       ├── store.js            # Session/category cache (never content source of truth)
│       ├── ui.js               # Icons, toasts, modals, skeletons, helpers
│       ├── components.js       # Header, drawer, pin cards, masonry, chips
│       ├── actions.js          # Delegated behaviours (save, like, delete, drawer…)
│       └── views/              # home, explore, search, pin, boards, profile, auth
├── api/
│   └── index.js                # Vercel serverless entry (wraps server/app.js)
├── server/                     # Backend
│   ├── app.js                  # Fastify factory — buildServer(), no port binding
│   ├── index.js                # Long-running entry: port, banner, graceful shutdown
│   ├── env.js                  # .env loading + Vercel/serverless defaults
│   ├── db.js                   # Turso/libSQL client resolution + helpers
│   ├── auth.js                 # bcrypt hashing + DB-backed sessions
│   ├── queries.js              # All SQL read/write operations
│   ├── routes.js               # REST API + validation + error handling
│   └── upload.js               # Image validation & storage
├── db/
│   ├── migrations/             # Ordered, tracked SQL migrations
│   ├── schema.sql              # Generated full-schema snapshot (read-only reference)
│   ├── migrate.js              # Migration runner
│   ├── seed.js                 # Idempotent seed data
│   ├── reset.js                # Rebuild the local database
│   └── export-schema.js        # Regenerate schema.sql from migrations
├── scripts/
│   ├── verify.js               # End-to-end verification suite (148 assertions)
│   └── db-query.js             # Read-only database inspector
├── vercel.json                 # Vercel: every route -> the single api/index.js function
├── .vercelignore               # Local files that are never uploaded
├── .env.example
├── .gitignore
└── package.json
```

---

## Turso setup

1. **Install the Turso CLI** (once):

   ```bash
   curl -sSfL https://get.tur.so/install.sh | bash
   turso auth signup        # or: turso auth login
   ```

2. **Create the database:**

   ```bash
   turso db create pinora
   ```

3. **Read the connection URL:**

   ```bash
   turso db show pinora --url
   # libsql://pinora-<organisation>.turso.io
   ```

4. **Create an auth token:**

   ```bash
   turso db tokens create pinora
   ```

5. **Put both values in `.env`** (never commit this file):

   ```env
   TURSO_DATABASE_URL=libsql://pinora-<organisation>.turso.io
   TURSO_AUTH_TOKEN=eyJhbGciOi...
   ```

6. **Create the tables and demo content:**

   ```bash
   npm run db:migrate
   npm run db:seed
   ```

Confirm which database the app is using at any time:

```bash
curl http://localhost:3000/api/health
# {"ok":true,...,"database":{"driver":"turso","label":"Turso remote database"}}
```

> **Offline / evaluation mode.** With no `TURSO_DATABASE_URL` (or `DB_DRIVER=local`) the
> exact same code and SQL run against a local libSQL file at `data/local.db`. The driver
> is reported by `/api/health`, so it is always obvious which database you are on.

---

## Environment variables

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `TURSO_DATABASE_URL` | for Turso | – | `libsql://…` URL of your Turso database |
| `TURSO_AUTH_TOKEN` | for Turso | – | Turso auth token (server-side only) |
| `DB_DRIVER` | no | `auto` | `auto` \| `turso` \| `local` |
| `DB_STRICT` | no | – | `1` disables the local fallback when Turso is unreachable |
| `DB_BOOTSTRAP` | no | – | `1` migrates + seeds an empty database on boot (always on in Vercel) |
| `LOCAL_DB_FILE` | no | `data/local.db` | Local libSQL file path |
| `UPLOAD_DIR` | no | `data/uploads` | Where uploaded images are written |
| `PORT` | no | `3000` | HTTP port |
| `HOST` | no | `0.0.0.0` | Bind address |
| `NODE_ENV` | no | `development` | `production` quiets logs |
| `APP_URL` | no | `http://localhost:3000` | Public base URL |

`.env` is listed in `.gitignore`. `TURSO_AUTH_TOKEN` is only ever read by
`server/db.js` — it is never embedded in HTML, JS bundles or API responses.

---

## Database initialisation

The database layer (`server/db.js`) resolves its target in this order:

1. `DB_DRIVER=local` → local libSQL file.
2. `DB_DRIVER=turso` → Turso only (fails loudly if unreachable).
3. `DB_DRIVER=auto` (default):
   - remote `TURSO_DATABASE_URL` present → connect to Turso;
   - connection fails → warn, then fall back to the local file (unless `DB_STRICT=1`).

The same `@libsql/client` library and identical SQL are used in both modes.

**Fresh install:**

```bash
npm install
cp .env.example .env      # add your Turso credentials
npm run db:setup          # migrate + seed
npm run dev
```

`npm run db:reset` rebuilds the **local** database from scratch (it refuses to touch a
remote Turso database).

---

## Migrations

Migrations are plain SQL files executed in filename order and recorded in the
`_migrations` table, so each file runs exactly once:

```
db/migrations/001_init.sql      -- tables, foreign keys, constraints, unique indexes
db/migrations/002_indexes.sql   -- listing/search/join indexes
db/migrations/003_triggers.sql  -- updated_at maintenance triggers
```

```bash
npm run db:migrate              # uses .env (Turso when configured)
npm run db:migrate -- --local   # force the local file
npm run db:migrate -- --turso    # force Turso
npm run db:schema                # regenerate db/schema.sql from the migrations
```

Adding a migration: create `db/migrations/004_your_change.sql` and run
`npm run db:migrate`. Use `IF NOT EXISTS` so a partially-applied run can be repeated.

`db/schema.sql` is a **generated** reference snapshot of the whole schema — never edit it
by hand; edit a migration and re-run `npm run db:schema`.

---

## Seed data

`npm run db:seed` is **idempotent** (deterministic primary keys + `INSERT OR IGNORE`), so
running it repeatedly never duplicates data.

It creates:

- **10 categories** — Fashion, Art, Food, Travel, DIY, Home Decor, Photography,
  Technology, Beauty, Design.
- **2 demo users** — `aisha_creates` / `marcus_designs`, password `Password1!`.
- **60 demo pins** across every category, with staggered timestamps so "recent"
  ordering is meaningful.
- **5 boards** (one private) with 19 board-pin relationships.
- **10 likes, 10 saves and 5 comments** so "trending" and "popular" genuinely differ
  from "recent".

### Demo logins

| Email | Username | Password |
| --- | --- | --- |
| `aisha@example.com` | `aisha_creates` | `Password1!` |
| `marcus@example.com` | `marcus_designs` | `Password1!` |

The login page has a **"Use demo credentials"** button. Users you create yourself are
stored in the same database and persist normally.

---

## Database schema

| Table | Purpose | Key columns |
| --- | --- | --- |
| `users` | Accounts & profiles | `id` PK, `username` UNIQUE, `email` UNIQUE, `password_hash`, `avatar_url`, `bio`, timestamps |
| `categories` | Content categories | `id` PK, `name` UNIQUE, `slug` UNIQUE, `description` |
| `pins` | Created content | `id` PK, `user_id` FK→users, `category_id` FK→categories, `image_url`, `destination_url`, `is_active` |
| `boards` | User collections | `id` PK, `user_id` FK→users, `name`, `cover_image_url`, `is_private` |
| `board_pins` | Board ↔ pin membership | `board_id` FK, `pin_id` FK, `saved_by_user_id` FK, `UNIQUE(board_id, pin_id)` |
| `pin_saves` | "This user saved this pin" | `UNIQUE(pin_id, user_id)` |
| `pin_likes` | Likes | `UNIQUE(pin_id, user_id)` |
| `pin_comments` | Comments | `pin_id` FK, `user_id` FK, `comment_text`, timestamps |
| `followers` | Social graph | `UNIQUE(follower_id, following_id)`, `CHECK (follower_id <> following_id)` |
| `user_sessions` | Server-side sessions | `id` PK, `user_id` FK, `expires_at` |
| `_migrations` | Applied migrations | `name` PK, `applied_at` |

**Constraints & indexes**

- Foreign keys with `ON DELETE CASCADE` (and `ON DELETE SET NULL` for `pins.category_id`).
- Unique constraints prevent duplicate likes, duplicate saves and duplicate board-pin
  records.
- `CHECK` constraints keep `is_active` / `is_private` in `{0,1}` and stop self-following.
- Indexes on `users.email`, `users.username`, `pins.user_id`, `pins.category_id`,
  `pins.created_at`, `pins.is_active`, `pins.title`, `boards.user_id`,
  `boards.is_private`, `board_pins.board_id`, `board_pins.pin_id`,
  `board_pins.saved_by_user_id`, `pin_saves.user_id`, `pin_saves.pin_id`,
  `pin_likes.*`, `pin_comments.*`, `followers.*`, `user_sessions.*`.
- Triggers keep `updated_at` current on `users`, `pins`, `boards` and `pin_comments`.

---

## Database relationships

```
users ──1:N──> pins ──N:1──> categories
users ──1:N──> boards
users ──1:N──> pin_saves / pin_likes / pin_comments / user_sessions
users ──N:M──> boards        (via board_pins.saved_by_user_id)
boards ──N:M──> pins         (via board_pins, UNIQUE(board_id, pin_id))
users ──N:M──> users         (via followers)
```

- One user creates many pins and many boards.
- One board holds many pins; one pin can appear on many boards.
- One pin can have many likes, saves and comments.
- One category contains many pins.
- Users can follow each other.

---

## Authentication

- **Signup** — full name, username, email, password, confirm password. Validates
  required fields, email format, uniqueness of email and username, minimum password
  length (8) and matching confirmation.
- **Login** — accepts **email or username** plus password.
- **Hashing** — `bcryptjs`, 11 salt rounds. Plain-text passwords are never stored or
  logged.
- **Sessions** — a 32-byte random id stored in the `user_sessions` table with a 30-day
  expiry, delivered as an httpOnly, SameSite=Lax cookie (Secure on HTTPS). Sessions are
  revoked on logout and expired rows are pruned hourly. Because sessions live in the
  database, they survive page reloads, browser restarts and server restarts.
- **Protected routes** — `/create`, `/boards`, `/settings` redirect guests to
  `/login?next=…`. API mutations return `401` without a valid session.
- **Authorisation** — every pin/board/comment mutation checks ownership server-side, so
  a user can only modify their own content, and private boards are never exposed to
  other users.

---

## Pin system

**Create** (`/create`): choose an image (file picker, drag & drop or paste a URL), add a
title, description, category, optional destination link and optionally save straight into
one of your boards. Validation: image required, title required, category required,
destination must be `http(s)://` when supplied.

**Edit** (`/pin/:id/edit`, owner only): title, description, category, destination and
image (re-upload or new URL).

**Delete** (owner only, behind a confirmation modal): the pin is soft-deleted
(`is_active = 0`) so it disappears from every feed, profile and listing while existing
board/save history stays intact.

**Detail page** (`/pin/:id`): large image, title, description, category badge, creator
card, like/save/share actions, live counts, destination link, comment thread and related
pins from the same category.

---

## Board system

- **Create** — name (required), description, privacy. Cover images are derived from the
  most recently saved pin unless an explicit cover is set.
- **Edit** — rename, description, cover and the private/public switch.
- **Delete** — behind a confirmation modal; the board and its board-pin rows are removed,
  the pins themselves are untouched.
- **Save a pin** — the Save button opens a board picker that marks the boards already
  containing the pin, and lets you create a new board inline. Saving writes a
  `board_pins` row plus a `pin_saves` row; duplicates are rejected both by the UI and by
  the `UNIQUE(board_id, pin_id)` constraint.
- **Remove a pin** — from a board detail page (confirmation modal). The `pin_saves` row is
  removed only when the pin is no longer in any of your boards.
- **Privacy** — private boards are visible only to their owner; other users receive `403`
  and never see them on a profile.

---

## Search

`GET /api/pins?q=…` performs a case-insensitive `LIKE` search across pin **title**,
**description**, **category name/slug** and **creator username/full name** — executed in
SQL, never by filtering a client-side array. Results include a total count; the UI shows
the query, the number of matches, sort options and a **Clear filters** control, plus
`"No pins found for your search."` when nothing matches.

---

## Filtering

- Category chips are generated from the `categories` table (with live pin counts).
- `GET /api/pins?category=<slug>` returns only that category; an unknown slug returns
  `404` rather than silently returning everything.
- **Search + category combine** (`?q=home&category=diy`) — both conditions are applied in
  one SQL statement.
- Sort modes: `recent`, `trending` (recency-weighted engagement) and `popular`
  (all-time engagement). Trending and popular are computed from likes, saves and
  comments in the database, so no pin is permanently "trending".

---

## Image storage approach

- **Turso stores metadata only** — `pins.image_url` holds a reference, never binary data.
- Uploaded files are written to `UPLOAD_DIR` (`data/uploads/` by default) and served from
  `/uploads/<uuid>.<ext>`.
- The server validates every upload: a real image signature (magic bytes) is required,
  only JPEG/PNG/WebP/GIF are accepted and the maximum size is **8 MB**. Client-supplied
  filenames and content types are never trusted.
- Uploads require authentication, and generated filenames are random UUIDs.
- Temporary browser object URLs (`URL.createObjectURL`) are only ever used for the local
  preview — the stored value is always a server URL.
- For production, point `UPLOAD_DIR` at a persistent volume, **or** swap
  `server/upload.js` for S3/R2/Cloudinary and keep storing the resulting URL in Turso
  (object storage stays separate from the database, as required).

---

## API reference

All endpoints are prefixed with `/api` and return JSON. Errors look like
`{ "ok": false, "error": "Friendly message" }` — never raw SQL or stack traces.

### Auth & account

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/api/auth/signup` | Creates an account and starts a session |
| `POST` | `/api/auth/login` | Email **or** username + password |
| `POST` | `/api/auth/logout` | Destroys the session |
| `GET` | `/api/auth/me` | Current user or `null` |
| `GET` | `/api/notifications` | Engagement on your own pins |
| `GET` | `/api/profile/me` | Own profile + stats |
| `PATCH` | `/api/profile/me` | Update full name, username, bio, avatar |
| `GET` | `/api/users/:username` | Public profile, pins, visible boards, follow state |
| `POST`/`DELETE` | `/api/users/:id/follow` | Follow / unfollow |

### Pins

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/api/feed` | Home feed: pins + trending + categories + stats |
| `GET` | `/api/discover` | Trending, popular, recent, categories, creators |
| `GET` | `/api/pins` | `q`, `category`, `sort`, `creator`, `saved=me`, `limit`, `offset` |
| `GET` | `/api/pins/trending` \| `/popular` \| `/recent` | Sort shortcuts |
| `GET` | `/api/pins/mine` | Your pins (auth) |
| `GET` | `/api/pins/saved` | Pins you saved (auth) |
| `GET` | `/api/pins/:id` | Detail + stats + comments + related |
| `POST` | `/api/pins` | Create (auth) |
| `PATCH` | `/api/pins/:id` | Update (owner) |
| `DELETE` | `/api/pins/:id` | Soft delete (owner) |
| `POST`/`DELETE` | `/api/pins/:id/like` | Like / unlike (auth) |
| `GET` | `/api/pins/:id/comments` | List comments |
| `POST` | `/api/pins/:id/comments` | Add comment (auth) |
| `DELETE` | `/api/comments/:id` | Delete own comment |
| `GET` | `/api/pins/:id/save-state` | Your boards + which contain this pin (auth) |

### Boards, categories, uploads

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/api/boards` | Your boards (auth) |
| `POST` | `/api/boards` | Create board (auth) |
| `GET` | `/api/boards/:id` | Board + pins (private boards owner-only) |
| `PATCH`/`DELETE` | `/api/boards/:id` | Update / delete (owner) |
| `POST` | `/api/boards/:id/pins` | Save a pin into a board (owner) |
| `DELETE` | `/api/boards/:id/pins/:pinId` | Remove a pin from a board (owner) |
| `GET` | `/api/categories` | Categories with pin counts |
| `GET` | `/api/categories/:slug` | Category + its pins |
| `POST` | `/api/uploads` | `multipart/form-data`, field `image` (auth) |
| `GET` | `/api/health` | Liveness + which database driver is active |

---

## Local development

```bash
npm install
cp .env.example .env          # fill in Turso credentials, or leave blank for local
npm run db:setup              # migrate + seed
npm run dev                   # http://localhost:3000  (node --watch)
```

Convenient extras:

```bash
npm run check                       # syntax-check every server module
npm run db:reset                    # rebuild the LOCAL database
npm run db:query -- "SELECT ..."    # read-only database inspector
```

The SPA has **no build step** — edit anything in `app/public` and reload.

---

## Build instructions

There is no bundler or compilation step; the frontend is native ES modules.

```bash
npm ci                    # reproducible install
npm run check             # syntax check
npm run verify             # full end-to-end verification (starts its own server)
NODE_ENV=production npm start
```

`npm start` serves both the API and the frontend. Uploads live in `UPLOAD_DIR` and the
database in Turso.

---

## Deployment instructions

### 1. Deploy to Vercel (what this repository ships)

`vercel.json` routes **every** request — static assets, `/api/*`, `/uploads/*` and
SPA deep links — to one serverless function, `api/index.js`, which wraps the exact
same Fastify app as `server/index.js`. No build step is required.

```bash
vercel link --yes --project pinora
vercel env add TURSO_DATABASE_URL production   # value is read from stdin
vercel env add TURSO_AUTH_TOKEN    production
vercel env add DB_STRICT           production  # =1: fail loudly, never fall back
vercel deploy --yes --prod                    # https://<project>.vercel.app
```

What the serverless layer adds (`server/env.js`, `server/app.js`):

- **Writable paths move to `/tmp`** — a Vercel function has a read-only filesystem
  outside `/tmp`, so `LOCAL_DB_FILE` and `UPLOAD_DIR` are redirected there.
- **Placeholder credentials are ignored** — with no usable `TURSO_*` values the app
  runs on the `/tmp` file instead of probing a non-existent host on every cold start.
- **Empty database ⇒ migrate + seed** on first boot (guarded by `SHOULD_BOOTSTRAP`),
  so a fresh instance is usable immediately.
- **Uploads are capped at 4 MB on Vercel** to return a friendly JSON error instead of
  the edge's 413 page (the platform rejects request bodies above 4.5 MB anyway).

> **Two Vercel caveats.** Uploaded images live in `/tmp` and therefore disappear when
> the instance is recycled — attach [Vercel Blob](https://vercel.com/docs/storage/vercel-blob)
> (or store an external image URL) for permanent uploads. And without `TURSO_*` set,
> each instance keeps its own `/tmp` database, so sessions and writes do not survive a
> second instance: always deploy with real credentials, as above.

### 2. Provision Turso

```bash
turso db create pinora
turso db show pinora --url
turso db tokens create pinora
```

### 3. Migrate and seed the remote database

Locally, with the production credentials in `.env`:

```bash
npm run db:migrate -- --turso
npm run db:seed    -- --turso     # optional: skip to ship an empty platform
```

### 4. Deploy the Node server (any long-running host)

Any Node host works (Fly.io, Render, Railway, a VPS, Docker…):

```bash
git clone <your-repo> && cd pinora
npm ci --omit=dev
```

Environment:

```env
TURSO_DATABASE_URL=libsql://pinora-<org>.turso.io
TURSO_AUTH_TOKEN=<token>
DB_DRIVER=turso
DB_STRICT=1
NODE_ENV=production
PORT=3000
APP_URL=https://your-domain.example
UPLOAD_DIR=/var/data/pinora-uploads
```

> **Important:** mount a persistent volume for `UPLOAD_DIR`, or move uploads to object
> storage — otherwise images disappear when the container is replaced.

Start command: `npm start` (or `node server/index.js`). Put the app behind HTTPS so
session cookies are sent with `Secure`.

### 5. Post-deploy checks

```bash
curl https://your-domain.example/api/health
# driver must be "turso"
```

Then sign up, create a pin, upload an image, save it to a board, refresh the page and
restart the app — everything must still be there.

---

## Verification & testing

`npm run verify` boots the whole stack hermetically: it creates a **fresh local libSQL
database**, runs migrations and seed, starts the real server on a private port, then
exercises **148 assertions** over HTTP and finally restarts the server to prove
persistence.

```
▶ Fresh database: migrate + seed          ▶ Authentication
▶ Server boot                             ▶ Image upload
▶ Public discovery (guest)                ▶ Pin creation, editing and deletion
▶ Search and category filtering           ▶ Boards and saving pins
▶ Likes and comments                      ▶ Notifications
▶ Profile management                      ▶ User data isolation
▶ Persistence across an application restart
▶ Pin deletion                            ▶ Security surface
=== 148 passed, 0 failed ===
```

Covered explicitly:

- signup validation (missing fields, invalid email, weak password, mismatched
  passwords, duplicate email, duplicate username)
- login by email and by username, wrong password, logout, re-login, session persistence
- image upload: valid upload served back, invalid bytes rejected, auth required
- pin create / edit / delete, validation, persistence, cross-user permissions
- search by title, description and category; combined search + category; category
  filters for Fashion, Art, Travel and DIY; result totals matching page contents
- boards: create, save pin, duplicate-save rejection, board contents, private-board
  protection, rename, privacy, delete, isolation between users
- likes (no double counting), comments (validation, ownership)
- notifications showing other users' activity but not your own
- **restart test** — pins, edits, sessions, profile changes, private boards and uploaded
  files all survive a server restart
- error responses never leak stack traces; `?q=' OR 1=1 --` is handled safely

---

## Security notes

- `TURSO_AUTH_TOKEN` is read only by the server and never serialised to the client.
- Passwords are bcrypt-hashed; `password_hash` is never returned by any endpoint.
- Sessions are httpOnly, SameSite=Lax, revocable and expire after 30 days.
- Every mutation re-checks ownership server-side (pins, boards, comments) — the UI hides
  controls, but the API is the real gate.
- Private boards return `403` for anyone but the owner, and are omitted from public
  profiles.
- Uploads are validated by magic bytes and capped at 8 MB; filenames are random UUIDs.
- SQL uses parameter binding everywhere (`?` placeholders) — no string concatenation.
- Errors return friendly messages; internal details are logged server-side only.
- `.env` and all `*.db` files are git-ignored.

---

## Accessibility & responsiveness

- Semantic landmarks (`header`, `main`, `nav`, `footer`, `article`), a skip link, and
  `aria-current` on the active navigation item.
- Dialogs use `role="dialog"`, `aria-modal`, labelled titles, Escape-to-close and a focus
  trap with focus restoration.
- Buttons carry text or `aria-label`s, toggles expose `aria-pressed`/`aria-expanded`, and
  images have descriptive `alt` text (decorative images use `alt=""`).
- Visible `:focus-visible` rings, 4.5:1+ contrast in light and dark themes.
- `prefers-reduced-motion` disables transforms and reveals card actions without hover.
- Breakpoints at 320 / 375 / 425 / 768 / 1024 / 1280 / 1440+ px, verified with **no
  horizontal overflow** at any width:
  - 320–767 px: 2-column masonry, compact header, hamburger drawer
  - 768–1023 px: 3-column masonry
  - 1024–1279 px: 4 columns, full desktop navigation
  - 1280–1599 px: 5 columns · 1600 px+: 6 columns

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Server logs `⚠️ Turso connection failed … falling back to the local libSQL file` | Placeholder or wrong Turso credentials | Put real values in `.env`, or set `DB_DRIVER=local` deliberately |
| `driver: "local"` in `/api/health` but you expected Turso | `TURSO_DATABASE_URL` missing/invalid, or `DB_DRIVER=local` | Correct `.env`; check with `turso db show <db> --url` |
| `Schema : MISSING` banner | Migrations never ran | `npm run db:migrate && npm run db:seed` |
| `Forbidden` when opening a board | The board is private and you are not the owner | Sign in as the owner |
| Uploads vanish after redeploy | `UPLOAD_DIR` is on ephemeral disk | Mount a volume or use object storage |
| `SQLITE_UNKNOWN: no such table` | Connected to an empty database | Run the migrations against that database |
| Seed appears to do nothing | It is idempotent — rows already exist | Expected; check `npm run db:query -- "SELECT COUNT(*) FROM pins"` |

---

## Licence

MIT — see `package.json`. Original implementation created for CWI Task 52.
