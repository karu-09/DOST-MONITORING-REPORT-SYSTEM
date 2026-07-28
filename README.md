# dost-records

Web-based document management + print-parity system for **DOST Regional Office No. 02**.

Each module is a split-pane admin tool: fill a form on the left, watch a live preview render on the right — and that preview must print pixel-identical to the office's existing paper form.

## Repo layout

```
.
├── dost-records/          # the actual Next.js app (see below)
├── dost-records-backup/   # backup snapshot, not the working app
├── CLAUDE.md              # project rules/conventions for AI-assisted dev
├── IMPLEMENTATION_STATUS.md
├── RECENT_IMPROVEMENTS.md
├── SPLIT_SCREEN_FORM_SPEC.md
├── TESTING_GUIDE.md
└── .kiro/specs/           # per-module requirements.md + tasks.md
```

All app code, dependencies, and scripts live inside `dost-records/`. Run every command from there.

## Stack

- **Next.js 14** (App Router) + **React 18** + **TypeScript**
- **Drizzle ORM** + **Postgres**
- **NextAuth** (Credentials provider, role-based access)
- **Tailwind CSS** + custom print CSS
- **Puppeteer** (PDF generation), **ExcelJS** (spreadsheet exports)

No automated test suite — QA is a manual checklist per module (print-parity, calculations, role gating). See `TESTING_GUIDE.md`.

## Getting started

```bash
cd dost-records
npm install
cp .env.local.example .env.local   # fill in DATABASE_URL, NEXTAUTH_URL, NEXTAUTH_SECRET
npm run db:push                    # or db:migrate, depending on your workflow
npm run db:seed                    # optional seed data
npm run dev
```

App runs at [http://localhost:3000](http://localhost:3000).

### Required environment variables

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection string |
| `NEXTAUTH_URL` | Base URL for NextAuth callbacks |
| `NEXTAUTH_SECRET` | Session encryption secret |
| `NODE_ENV` | `development` / `production` |

### Scripts

| Command | Does |
|---|---|
| `npm run dev` | Start dev server |
| `npm run build` | Production build |
| `npm run start` | Run production build |
| `npm run lint` | Lint |
| `npm run db:generate` | Generate Drizzle migration from schema |
| `npm run db:migrate` | Apply migrations |
| `npm run db:push` | Push schema directly (dev) |
| `npm run db:studio` | Open Drizzle Studio |
| `npm run db:seed` | Seed the database |

## Modules

Each module lives at `app/(app)/<module-slug>/` with a matching print template in `src/templates/<module-slug>.ts`.

- Application for Leave
- Attendance
- Cash Advance
- Certificate of Appearance
- Daily Time Record
- Incident Report
- Inspection Report
- Maintenance Request
- Meeting Minutes
- Office Memorandum
- Overtime Application
- Pass Slips
- Property Acknowledgement
- Property Gate Pass
- Purchase Request
- Requisition Slip
- Service Record
- Travel Order
- Vehicle Trip Ticket

Some modules mirror real DOST paperwork requiring signatures (e.g. Travel Order, Cash Advance) and carry an approval chain + immutable-after-submit state. Others (e.g. Petty Cash Voucher, Fuel Withdrawal Slip) are single-fill forms with no routing. Check each module's `.kiro/specs/<module>/requirements.md` for its specific rules.

## Conventions

- Route: `app/(app)/<module-slug>/`
- Print/PDF template: `src/templates/<module-slug>.ts`
- Shared UI: `src/components/`
- DB schema: `src/db/schema.ts` (Drizzle)

Global invariants (viewport lock, print engine CSS, fill-in-the-blank styling, split-pane layout) are documented in `CLAUDE.md` — read that before touching layout or print code.

## Build process

New modules follow a fixed cycle:

1. **Spec** — `.kiro/specs/<module>/requirements.md` (fields, calc rules, validation, approval chain needed?)
2. **Schema/API** — Drizzle table + route + migration
3. **Form + print template** — split-pane form, live preview, print CSS
4. **QA pass** — print-parity check, calc check, role check
5. **Done** — update `.kiro/specs/<module>/tasks.md`

One module at a time.

## Docs

- [`CLAUDE.md`](CLAUDE.md) — invariant rules, conventions, per-module workflow
- [`TESTING_GUIDE.md`](TESTING_GUIDE.md) — manual QA checklist
- [`SPLIT_SCREEN_FORM_SPEC.md`](SPLIT_SCREEN_FORM_SPEC.md) — split-pane layout spec
- [`RECENT_IMPROVEMENTS.md`](RECENT_IMPROVEMENTS.md) — changelog of recent fixes
- [`IMPLEMENTATION_STATUS.md`](IMPLEMENTATION_STATUS.md) — status of in-flight feature work
