# Codex Task List — Theory Tab Player (V1)

> Scope anchor: V1 delivers a web player with **YouTube embed**, a **music library**, and a **synced theory timeline** with blocks (degrees 1–7 + roman numerals), plus a **simple internal CRUD** to register the first tabs.  
> Use the repo structure already defined (`apps/api`, `apps/web`, `docs/architecture`).  
> Keep the architecture rules: modules by business area, strict layer separation, no “shared as trash”.  

---

## 0) Project bootstrap

### TASK-0001 — Initialize monorepo + base tooling
- **Goal:** Create a runnable repository skeleton for API + Web.
- **Deliverables:**
  - `apps/api/` + `apps/web/` scaffolds
  - root scripts: install, dev, build, test, lint
  - `.env.example` for both apps
  - `docs/architecture/` folder created
- **Acceptance criteria:**
  - `pnpm dev` (or repo’s chosen runner) starts API + Web locally without errors
  - CI-friendly commands exist: `lint`, `test`, `build`

### TASK-0002 — Create the folder structure exactly as the architecture
- **Goal:** Materialize module folders + shared folders (backend + frontend).
- **Deliverables:** folder trees under:
  - `apps/api/src/shared/*`
  - `apps/api/src/modules/{identity,music-catalog,theory-tabs}/*`
  - `apps/web/src/shared/*`
  - `apps/web/src/modules/{auth,library,player,admin}/*`
- **Acceptance criteria:**
  - No file exceeds folder depth limits from `src/`
  - No empty “fake” layers (if no code yet, add placeholders + TODO)

---

## 1) Database + migrations

### TASK-0101 — Define DB schema (5 core entities) + migrations
- **Goal:** Implement schema for: `usuario`, `musica`, `theorytab`, `trilha_instrumento`, `bloco_teorico`.
- **Deliverables:**
  - DB schema file(s) + migration scripts
  - constraints + indexes:
    - `usuario.email` unique
    - `theorytab.musica_id` unique (V1 1–1)
    - `(bloco_teorico.trilha_id, bloco_teorico.inicio_seg)` index
- **Acceptance criteria:**
  - Fresh DB can be created from scratch via one command
  - Migration is deterministic and repeatable

### TASK-0102 — Seed data (minimum usable V1)
- **Goal:** Seed: 1 admin user + ~10–20 musics + at least 1 theory tab with 1 instrument track + blocks.
- **Deliverables:**
  - `seed` script runnable locally
  - sample dataset committed (sql/json/ts — consistent with stack)
- **Acceptance criteria:**
  - After seeding, the Library page can list music and Player endpoint returns at least one working track

---

## 2) API foundation (shared)

### TASK-0201 — HTTP server bootstrap + routing + healthcheck
- **Goal:** Have an API server with versioned routes and a health endpoint.
- **Deliverables:**
  - `GET /health` returns `{ ok: true }`
  - centralized route registration (`routes.ts`)
- **Acceptance criteria:**
  - Server starts clean, logs port, and healthcheck returns 200

### TASK-0202 — Error handling + Result pattern
- **Goal:** Standardize error responses and internal Result type.
- **Deliverables:**
  - `shared/errors/*` for typed errors
  - `shared/utils/result.*` implementing `Result.ok` / `Result.fail`
  - global error middleware mapping domain errors → HTTP status
- **Acceptance criteria:**
  - Any thrown/returned domain error becomes a consistent JSON response structure

### TASK-0203 — Request validation + DTO conventions
- **Goal:** Ensure controllers validate input format and map to DTOs.
- **Deliverables:**
  - DTO validation helper(s) in shared
  - a convention doc in `docs/architecture/CONVENTIONS.md`
- **Acceptance criteria:**
  - Invalid inputs return 400 with useful message and error code

### TASK-0204 — Auth middleware (session/token) + role guard
- **Goal:** Protect admin endpoints; allow public endpoints.
- **Deliverables:**
  - `shared/middlewares/auth.middleware.*`
  - `requireRole('admin')` guard
- **Acceptance criteria:**
  - `/admin/*` endpoints reject unauthenticated requests (401)
  - `/admin/*` endpoints reject non-admin users (403)

---

## 3) Identity module (MOD-001)

### TASK-0301 — Identity entities + repository
- **Goal:** Implement `User` entity mapping and persistence.
- **Deliverables:**
  - `modules/identity/entities/user.entity.*`
  - `modules/identity/interfaces/user-repository.interface.*`
  - `modules/identity/repository/user.repository.db.*`
- **Acceptance criteria:**
  - repository supports: `findByEmail`, `findById`, `create`

### TASK-0302 — Login / Me / Logout use cases + controllers
- **Goal:** Provide basic authentication flows for V1.
- **Deliverables:**
  - services: `login`, `get-me`, `logout`
  - controllers: `/auth/login`, `/auth/me`, `/auth/logout`
- **Acceptance criteria:**
  - Login returns token/session
  - `/auth/me` returns current user
  - Logout invalidates session/token

---

## 4) Music Catalog module (MOD-002)

### TASK-0401 — Music entity + repository
- **Goal:** Persist and query music catalog.
- **Deliverables:**
  - `Music` entity
  - repository methods: `create`, `update`, `findById`, `search(query)`
- **Acceptance criteria:**
  - Search supports title/artist (case-insensitive)
  - Data returned includes youtube reference (id/url)

### TASK-0402 — Public endpoints: list/search + detail
- **Goal:** Provide the Library UI data needs.
- **Deliverables:**
  - `GET /musicas?search=...`
  - `GET /musicas/:id`
- **Acceptance criteria:**
  - List endpoint returns `availableInstruments` derived from tracks (join/aggregate)
  - Endpoints are stable and documented in `docs/api.md`

### TASK-0403 — Admin endpoints: create/update music
- **Goal:** Allow internal CRUD for music.
- **Deliverables:**
  - `POST /admin/musicas`
  - `PATCH /admin/musicas/:id`
- **Acceptance criteria:**
  - Protected by admin role
  - Validates youtube_id/url format and required fields

---

## 5) Theory Tabs module (MOD-003)

### TASK-0501 — Entities + repositories (TheoryTab, Track, Block)
- **Goal:** Implement persistence for theory content.
- **Deliverables:**
  - entities:
    - `theory-tab.entity.*`
    - `instrument-track.entity.*`
    - `theoretical-block.entity.*`
  - repositories:
    - theory tab repo
    - track repo
    - block repo (bulk operations)
- **Acceptance criteria:**
  - blocks can be listed ordered by `inicio_seg`
  - track uniqueness enforced per V1 rule

### TASK-0502 — Admin: create theory tab + change status
- **Goal:** Lifecycle support for V1.
- **Deliverables:**
  - `POST /admin/theory-tabs`
  - `PATCH /admin/theory-tabs/:id/status`
- **Acceptance criteria:**
  - Allowed status values enforced
  - Status transitions validated (V1: simple rules OK)

### TASK-0503 — Admin: upsert instrument track
- **Goal:** Add/modify a track for an instrument.
- **Deliverables:**
  - `POST /admin/theory-tabs/:id/trilhas` (create)
  - `PATCH /admin/trilhas/:id` (update fields)
- **Acceptance criteria:**
  - `instrumento` enum validated
  - `ativo` toggling works

### TASK-0504 — Admin: replace blocks in bulk (idempotent)
- **Goal:** Efficiently load blocks for a track.
- **Deliverables:**
  - `PUT /admin/trilhas/:id/blocos` accepting an array of blocks
  - repository supports `replaceAll(trackId, blocks[])` in one transaction
- **Acceptance criteria:**
  - Old blocks are replaced fully and atomically
  - Validation ensures `inicio_seg >= 0`, `duracao_seg > 0`

### TASK-0505 — Player endpoint (public): music + instrument → track + blocks
- **Goal:** Provide the Player UI everything it needs in one call.
- **Deliverables:**
  - `GET /musicas/:id/player?instrumento=piano`
  - response includes:
    - music metadata + youtube_id
    - selected track metadata
    - ordered blocks
- **Acceptance criteria:**
  - Returns 404 if no track for instrument
  - Returns blocks ordered and ready for time-based highlighting

---

## 6) Admin tool (simple internal CRUD)

### TASK-0601 — Admin UI page(s) OR Admin CLI scripts
- **Goal:** Create/edit music + theory tabs + tracks + blocks without editing code.
- **Deliverables (choose one path):**
  - **Option A (UI):** `apps/web/src/modules/admin/*` screens + forms
  - **Option B (CLI):** `apps/api/scripts/*` (create music/tab/track/blocks)
- **Acceptance criteria:**
  - You can create a full playable setup (music + tab + track + blocks) end-to-end

### TASK-0602 — Minimal “bulk import” format
- **Goal:** Speed up content creation.
- **Deliverables:**
  - Define JSON format for blocks and provide example file(s)
  - Admin UI/CLI supports uploading/reading that format
- **Acceptance criteria:**
  - Import creates blocks in correct order and time ranges

---

## 7) Web app foundation

### TASK-0701 — Web routing + layouts
- **Goal:** Pages: auth, library, player, admin.
- **Deliverables:**
  - route config
  - base layout components
- **Acceptance criteria:**
  - Direct navigation to each route works
  - Auth-gated admin route redirects when not admin

### TASK-0702 — API client + typed models
- **Goal:** Centralize API calls and types.
- **Deliverables:**
  - `shared/api/*` client
  - `shared/types/*` for DTO types (sync with backend)
- **Acceptance criteria:**
  - No fetch calls scattered in components; all go through the client

---

## 8) Auth UI (V1)

### TASK-0801 — Login screen + session storage
- **Goal:** Basic login for admin; optional guest mode.
- **Deliverables:**
  - login form
  - token/session persistence
  - logout button
- **Acceptance criteria:**
  - Refresh keeps user logged in
  - Logout clears session

---

## 9) Library UI (V1)

### TASK-0901 — Library page (list + search)
- **Goal:** Browse and search musics.
- **Deliverables:**
  - library list UI
  - search input with debounce
  - show `availableInstruments`
- **Acceptance criteria:**
  - Search queries API and updates list
  - Click a music navigates to Player

---

## 10) Player UI (V1)

### TASK-1001 — Player page shell + YouTube embed
- **Goal:** Render embedded YouTube video for the selected music.
- **Deliverables:**
  - Player route: `/player/:musicId`
  - embed component that can expose `currentTime` (polling or API)
- **Acceptance criteria:**
  - Video loads and plays/pauses from UI controls

### TASK-1002 — Instrument selector + track loading
- **Goal:** Switch instrument and reload track data.
- **Deliverables:**
  - instrument dropdown/tabs
  - fetch `/musicas/:id/player?instrumento=...`
- **Acceptance criteria:**
  - Changing instrument updates timeline blocks and labels immediately

### TASK-1003 — Timeline rendering (blocks 1–7 + roman numerals)
- **Goal:** Show theory blocks on a timeline.
- **Deliverables:**
  - timeline component
  - block rendering with:
    - `grau` (1–7)
    - `funcao_roman` (I–VII for chords)
    - optional `nota_letra`
- **Acceptance criteria:**
  - Blocks display with correct time spans (start + duration)
  - Blocks are visually distinct by degree (color mapping in UI)

### TASK-1004 — Sync engine: highlight active block(s) by currentTime
- **Goal:** During playback, highlight the currently active block.
- **Deliverables:**
  - sync loop (requestAnimationFrame or interval)
  - efficient lookup (pointer/index, not O(n) each tick)
- **Acceptance criteria:**
  - Highlight updates smoothly
  - No frame drops on typical block counts

### TASK-1005 — Basic playback controls
- **Goal:** Play/pause, seek, and jump to next/prev block.
- **Deliverables:**
  - play/pause button
  - seek bar (optional) or block-click-to-seek
  - next/prev block controls
- **Acceptance criteria:**
  - Clicking a block seeks video to that block’s start time

---

## 11) Cross-cutting: docs, tests, QA

### TASK-1101 — API documentation (contracts)
- **Goal:** Document endpoints and DTO shapes.
- **Deliverables:**
  - `docs/api.md` with examples for:
    - list musics
    - player endpoint
    - admin CRUD
- **Acceptance criteria:**
  - A developer can implement frontend calls just from this doc

### TASK-1102 — Minimal automated tests
- **Goal:** Ensure core flows don’t break.
- **Deliverables:**
  - API tests for:
    - auth
    - list/search musics
    - player endpoint data shape
  - Frontend smoke test (route renders)
- **Acceptance criteria:**
  - `test` command passes on CI

### TASK-1103 — Lint + formatting + pre-commit
- **Goal:** Keep repo consistent.
- **Deliverables:**
  - lint config + format config
  - pre-commit hook (optional)
- **Acceptance criteria:**
  - CI fails on lint errors

### TASK-1104 — Release packaging (local deploy)
- **Goal:** One-command run for reviewers.
- **Deliverables:**
  - docker compose (or equivalent) for DB + API + Web
  - README with setup steps
- **Acceptance criteria:**
  - Fresh machine can run project with minimal steps

---

## 12) Definition of Done (V1 delivery gate)

Project is “done” when:

- ✅ Library lists and searches musics
- ✅ Player loads YouTube embed and theory blocks for chosen instrument
- ✅ Timeline highlights blocks in sync with playback time
- ✅ Admin can create music + theory tab + track + blocks end-to-end
- ✅ Seed provides at least one fully working example track
- ✅ Basic tests + docs + one-command local run exist

