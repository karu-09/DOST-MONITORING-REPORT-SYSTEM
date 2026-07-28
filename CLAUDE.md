# CLAUDE.md — dost-records

Web-based document management + print-parity system for DOST Regional Office No. 02. Admin tool: fill form left pane, live preview right pane, preview must render pixel-identical to physical Letter-sized printout.

## Stack

- Next.js 14 App Router, React 18, TypeScript
- Drizzle ORM + Postgres
- NextAuth (Credentials provider, `role` field on user)
- Tailwind + custom CSS (Grid/Flexbox for print layouts)
- Puppeteer (PDF generation), ExcelJS (exports)
- App lives in `dost-records/`. No test runner installed — QA is manual checklist, not automated suite.

## Global invariant rules (do not re-derive, do not violate)

**Main menu viewport lock**
Wrapper: `height: 100vh; width: 100vw; overflow: hidden; box-sizing: border-box;`, body margins stripped. Zero scroll. Pill-shaped flex cards, thick black border, `border-radius: 9999px`, icon square left + stacked title/description right, `flex: 1` to shrink-to-fit.

**Print engine**
Preview wrapper enforces Letter size: `width: 8.5in; min-height: 11in;`.
Kill browser print header/footer stamp:
```css
@media print {
  @page { margin: 0 !important; }
  body { margin: 1cm; }
}
```
`window.print()` for print trigger, always pair with `window.onafterprint` to clear loading/button-lock state — button must not stay locked after one print.

**Fill-in-the-blank text alignment**
```css
display: inline-block;
border-bottom: 1px solid black;
text-align: center;
line-height: 1;
padding-bottom: 2px;
white-space: normal;
word-wrap: break-word;
```

**Layout**: every module page is split-pane — left scrollable form (fields, dropdowns, dynamic grids, "Save & Print" + "Full View" actions), right scrollable wrapper holding the live preview document. Don't deviate per-module without reason.

## Conventions

- Route: `app/(app)/<module-slug>/`
- Print/PDF template: `src/templates/<module-slug>.ts`
- Shared UI: `src/components/`
- DB: `src/db/schema.ts` (Drizzle)

## Roles (verbal hats, not files — solo dev)

Switch explicitly in prompts: "Acting as X, do Y."

- **Schema/API** — Drizzle table, route, migration
- **Frontend** — split-pane form, live preview, print CSS
- **Security** — role/session check, input validation, secrets. Invoke per module.
- **QA** — manual checklist: print-parity, calc correctness, role gating. No automated suite exists.
- **Refactor** — dedupe across modules, no-behavior-change. Run every 3-4 modules — 19 near-identical modules means copy-paste drift is the top risk, not the last one you catch.

## Per-module build cycle

Foundation (auth, DB, CI-equivalent, first 3 modules) is done. For each remaining module:

1. Spec — `.kiro/specs/<module>/requirements.md` (fields, calc rules, validation, does it need an approval chain?) — you approve before code
2. Schema/API
3. Form + print template
4. QA pass (print-parity check against source form, calc check, role check)
5. Done — update `.kiro/specs/<module>/tasks.md`

**Approval-chain check per module** — don't assume uniform. Simple single-fill forms (Petty Cash Voucher, Fuel Withdrawal Slip) skip routing. Forms mirroring real DOST paperwork with signatures (Travel Order, Cash Advance, Liquidation Report, Certificate of Travel Completed) likely need approver role + immutable-after-submit state — flag in that module's requirements.md, don't skip by default.

## Task prompt template

```
Module: <name>
User Story: As a [role], I want [action], so that [benefit].
Acceptance Criteria: 1. 2. 3.
Current State → Expected State: [before] → [after]
Permissions: view/edit/approve — [only if module has approval chain]
Affected: Frontend / Backend / Database
Security: [access + validation rules]
Testing: [print-parity, calc, role checks needed]
Constraints:
- Do not change unrelated modules.
- Do not bypass role checks.
- Do not overwrite submitted data.
- Do not commit secrets.
```

One module at a time. Never "build the whole feature."

## Risks

| Risk | Mitigation |
|---|---|
| Print CSS / DOST header drift across 19 modules | Refactor pass every 3-4 modules |
| Scope creep | requirements.md approved before code |
| Wrong/missing approval routing | Explicit per-module check, not blanket skip |
| Data loss | DB backups, tested restore before go-live |
