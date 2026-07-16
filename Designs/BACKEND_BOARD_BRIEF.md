# Claude Design Brief — T Academy Backend Board (interactive HTML)

**Deliverable:** one self-contained interactive HTML file —
`Designs/T Academy - Backend.dc.html` — that visualizes the entire backend
specification so Ash can review it, flag decisions, and sign off before the build.
It is a **working review instrument**, not a poster: every claim in the spec should
be explorable, and the open decisions must be answerable inside the page.

**Source of truth:** `BACKEND_SPEC.md` in this repo. Do not invent schema, policies,
or numbers — everything rendered must come from that document. Embed the content as
a single JSON data object inside the file (tables, columns, FKs, policies, flows,
decisions) so the visuals are generated from data, not hand-drawn per diagram.

---

## 1. Format and conventions (match the existing comps)

- Single HTML file, fully self-contained: no CDN scripts, no external fonts or
  images, no network requests. Inline all CSS/JS. It must open from disk.
- Follow the board grammar of `T Academy - App.dc.html`: a header strip
  ("T Academy · backend · the complete system, one board"), then **numbered
  sections** (`01 · …`), each with a one-line annotation caption explaining the
  design intent — same voice as the app comp (declarative, no marketing).
- Desktop-first (this is a review tool), readable at 1280px; must not horizontally
  scroll the page — wide diagrams scroll inside their own container.
- Light theme only, matching the TEF system.

## 2. Design system (non-negotiable)

| Token | Value |
|---|---|
| Background / surface | `#FFFFFF` / `#FAFAFA` |
| Ink / muted | `#0D0D0D` / `#525252`, `#8A8A8A` |
| Hairline | `rgba(0,0,0,.10)` |
| Accent (single) | `#185FA5` — links, active states, flow highlights |
| Status ok / warn / bad | `#1E7A46` / `#9A6A00` / `#B03A2E` — RLS allow/conditional/deny, met/unmet criteria |
| Type | Geist → fall back `system-ui, -apple-system, sans-serif`; **Geist Mono → `ui-monospace`** for every identifier, column name, enum value, and number |
| Radius / spacing | 10–16px radius, generous whitespace, hairline borders — near-monochrome; color only where it means something |

Status colors never stand alone: pair each with a glyph or word (`✓ allow`,
`~ conditional`, `× deny`), exactly like the app's verdict rule.

## 3. Board sections (in order)

**01 · SYSTEM MAP** — the architecture diagram (spec §1): app, Supabase (Postgres /
Auth / Edge Functions / Vault / cron / Storage), Expo push, widgets, the website
seam. Hovering a node highlights its connections and shows a two-line description;
clicking pins it. The bridge seam is visually distinct (the one place two backends
touch).

**02 · SCHEMA EXPLORER** — the centerpiece. An interactive ER diagram of all 20
tables (7 inherited, 13 new — badge each `inherited` / `new`).
- Tables are cards: name, one-line purpose, key columns (PK/FK marked, types in
  mono). Click → a right-side detail panel with the full column list, indexes,
  constraints, triggers touching it, and its RLS policies verbatim.
- FK edges drawn between cards; hovering an edge shows the relationship reading
  ("gates n─1 gate_templates").
- **Role lens** — a segmented control: `anon · student · tutor · service`. Selecting
  a role dims every table/operation that role cannot touch and annotates the rest
  with its allowed operations. This is the fastest way to *see* the security model.
- Domain filter chips: `journey · content · telemetry · flashcards · gates · notifications`.

**03 · RLS MATRIX** — the policy matrix from spec §4 as a grid (tables × roles ×
select/insert/update/delete), cells colored ok/warn/bad with the glyph rule. Click a
cell → the exact policy condition. Note the two load-bearing rows (append-only
telemetry, booking-only gate update) with callouts.

**04 · TELEMETRY FLOW** — the event stream (spec §5): producers → `telemetry_events`
→ triggers (stamp/clamp/rollup) → derived numbers (cadence strip, pace, streak, gate
evidence). Include a **simulator**: a week-of-events sample the reviewer can edit
(add a set completion, change minutes) with the derived hours/pace/gate-ETA
recomputing live using the spec's exact formulas. Show the clamp visibly (enter 300
minutes on a 20-minute set → watch it clamp).

**05 · GATE STATE MACHINE** — the six states and transitions (spec §6) as a
horizontal machine; click a transition to see who/what fires it (student, tutor,
`evaluate-gates`) and the criteria JSON ↔ evidence JSON side by side with met/unmet
status colors. Include the R5 exam-cockpit special case.

**06 · NOTIFICATION SIMULATOR** — the five push types and the eligibility engine
(spec §11). Controls: local hour slider, per-type pref toggles, daily-cap counter,
assignment due-in hours, streak state. Output: which pushes fire this sweep, which
are suppressed and **why** (quiet hours / cap / dedup / pref), rendered as
lock-screen-style push cards like app board 32.

**07 · SRS SIMULATOR** — SM-2 (spec §7.2). A card with the four grade buttons
showing their computed next intervals (as in app board 16); pressing grades steps
the state (interval, ease, reps, lapses) and appends to a visible history table.
A caption notes the FSRS-later decision and the offline idempotency story.

**08 · WEBVIEW BRIDGE** — the four-step handshake (spec §10) as a sequence diagram
(app → mint-web-session → POST /api/app-bridge → cookie → /programs). Step-through
control; annotate the two security properties (no token in URLs, 5-minute exp +
jti). Mark the shared secret's two homes (Vault, Vercel env).

**09 · SERVER SURFACE** — inventory cards for the 5 Edge Functions, 5 RPCs, 6
triggers, 2 cron jobs (spec §8). Each card: trigger/caller, one-line behavior,
what it reads/writes (chips linking back to the schema explorer).

**10 · MIGRATIONS & SEEDS** — the 9-migration ladder (spec §14) as an ordered rail
with per-step contents; seed order beneath.

**11 · DECISIONS** — the review mechanism, and the reason this board exists:
- The 8 held decisions (D-1…D-8) as cards with an `agree / revisit` toggle.
- The 5 open questions (O-1…O-5) as cards with their options as radio choices plus
  a free-text note field.
- State persists in `localStorage`; a sticky **"Export review"** button copies a
  JSON summary `{decisions, answers, notes, timestamp}` to the clipboard — that
  export is what feeds the backend finalization pass.

## 4. Interactions (global)

- A left rail table-of-contents (mirrors the app comp's board index); active section
  tracks scroll.
- Everything keyboard-reachable; panels dismiss on `Esc`; focus states visible.
- All state (lens, filters, simulators, decisions) survives reload via
  `localStorage` under a single `tacademy-backend-board` key.
- No animation beyond 150–200ms transitions; respect `prefers-reduced-motion`.

## 5. Acceptance checklist

- [ ] Opens from disk, zero network requests, one file
- [ ] All 20 tables, every RLS policy, all 5 push types, all 9 migrations present and traceable to `BACKEND_SPEC.md`
- [ ] Role lens correctly reflects the §4 matrix for all four roles
- [ ] Simulators implement the spec's exact formulas (SM-2 numbers, pace grace hour, clamp rules, quiet-hours wrap at midnight)
- [ ] Decisions panel exports valid JSON; state survives reload
- [ ] Monochrome + `#185FA5` held; status colors always paired with glyph/word
- [ ] No horizontal page scroll at 1280px

---

*Feed this brief together with `BACKEND_SPEC.md` to Claude Design. The output file
lands in `Designs/` beside the app comp.*
