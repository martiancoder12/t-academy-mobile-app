# T Academy — Companion App (TEF Exam Prep Academy)

The native iOS + Android companion app for **TEF Exam Prep Academy** (tefexamprep.ca),
built by **re-skinning the French with Ash companion app**. The engineering — the
assignment loop, the exercise player, the notification engine, the Supabase security
model — carries over intact. The brand, the product framing, and several new surfaces
change: *French with Ash* becomes *T Academy*, and the app's spine becomes the
**five-release roadmap** — where the student is, what today contributed, and what
evidence the next gate still needs.

> **Status: pre-build.** The design comp is complete (38 boards), the reference
> codebase is in place, and the new app has not been scaffolded yet. This README is
> the starting brief.

## What's in this folder

```
BACKEND_SPEC.md                The complete backend build spec: schema (20 tables),
                               RLS matrix, telemetry engine, gate state machine,
                               SRS scheduling, notifications, webview bridge,
                               migrations, and the open decisions.
Designs/
  T Academy - App.dc.html      The full design comp: 35 screens + a home-screen
                               widget set, first run → exam booking, student and
                               tutor sides, annotated board by board.
  BACKEND_BOARD_BRIEF.md       Brief for Claude Design: render BACKEND_SPEC.md as
                               an interactive HTML review board (schema explorer,
                               role lens, simulators, decision sign-off).
Reference codebase (Frenchwithash)/
  fwa-app/                     The app being re-skinned (code-complete v1).
                               Its README, DEVLOG.md, and docs/ are the deep
                               reference for everything inherited.
  PRD_French_with_Ash_Mobile_App.md            Product spec (AF-1…AF-14)
  Technical_Implementation_Plan_Mobile_App.md  Build spec
```

> **Note:** `Reference codebase (Frenchwithash)/` is a local working clone of
> [`martiancoder12/frenchwithash-app`](https://github.com/martiancoder12/frenchwithash-app)
> (private) and is **gitignored** — it has its own git history and contains live
> credentials (`.env`, `google-services.json`), so it is never committed here.

Related project: the TEF Exam Prep Academy **website** lives at
`~/Documents/Doing/TEF Exam Prep Academy` — its `public/assets/css/tef.css` is the
canonical design system this app adopts, and its program pages are what the app's
Programs tab renders in a webview.

## The core loop (inherited, reframed)

The FWA loop survives unchanged underneath:

```
Tutor assigns a set (due date + note) → student gets a push
   → works the drills offline-capable, accent-aware checking
   → marks complete → tutor sees completion on the roster
Between sessions: due-soon / overdue / streak nudges,
   timezone-aware, quiet hours, daily cap.
```

What T Academy adds on top is the framing: **every completed task becomes a
telemetry line** (hours banked, words written, cards reviewed, listening minutes),
telemetry accumulates into **evidence**, and evidence — not the calendar — unlocks
the **gate** to the next release (R1 → R5, ending in the exam cockpit and TEF
booking). Four tabs: **home · programs · flashcards · profile**.

## What carries over unchanged

| Layer | Choice (from fwa-app) |
|---|---|
| App | Expo (React Native) SDK 56, managed workflow, EAS |
| Language | TypeScript (strict) |
| Navigation | Expo Router (file-based, typed routes), role-gated student/tutor trees |
| Server state / offline | TanStack Query + AsyncStorage persistence + NetInfo |
| Backend | Supabase — Postgres, Auth (magic-link/PKCE), RLS, Edge Functions, Vault, pg_cron |
| Validation | Zod exercise schema (the contract with the content pipeline) |
| Secure storage | expo-secure-store (Keychain / Keystore) |
| Push | Expo Notifications + Expo push service |
| Tests | Vitest (judge, content, streak, reminders + live RLS integration) |

Also inherited as-is: invite-only magic-link auth, the accent-aware judge
(`correct` / `right-but-accents` / `wrong`), the full player (gap-fill, choice,
matching, production, conjugation), offline cache + sync queue, the nudge engine
(timezone, quiet hours 21:00–08:00, daily cap, dedup), streaks, the content bridge
(markdown → Zod-validated exercise JSON), and the database-enforced security model
(RLS on every table, no privilege escalation, service key never ships).

## The re-skin

### Design system: FWA brand → TEF monochrome

The warm FWA identity is replaced wholesale by the TEF system (near-monochrome,
Geist, one blue accent) already shipped on the website:

| Token | French with Ash | T Academy |
|---|---|---|
| Background | paper `#FCFAF5` / cream `#F8F4ED` | white `#FFFFFF` / surface `#FAFAFA` |
| Primary text | navy `#1A2C4E` | ink `#0D0D0D` |
| Accent | coral `#D97757` | blue `#185FA5` (single accent) |
| Headings / French text | Lora (serif) | Geist |
| Body / UI | Work Sans | Geist; Geist Mono for telemetry numbers |
| Verdict ok | green `#1E7A46` | **kept** |
| Verdict accents-only | amber `#9A6A00` | **kept** |
| Verdict wrong | red `#B03A2E` | **kept** |

The green/amber/red verdict tones are a deliberate exception to the monochrome
system; verdicts never rely on color alone (icon + text always pair with it).
Source of truth: `tef.css` in the website repo, and the comp in `Designs/`.

### Identity swap (app.json / EAS / stores)

| Field | From | To |
|---|---|---|
| `expo.name` | French with Ash | T Academy |
| `expo.slug` | french-with-ash-mobile-app | *(new — e.g. t-academy-app)* |
| `expo.scheme` | `fwaapp` | *(new — e.g. `tacademy`)* |
| Bundle id / package | `com.martiancoder.frenchwithash` | *(new)* |
| EAS project | `@martiancoder/french-with-ash-mobile-app` | *(new project)* |
| Icons / splash / adaptive icon | FWA assets | T monogram, monochrome set |
| Push copy | "New set from Ashfaaq…" | T Academy voice (see comp boards 06, 32) |
| Supabase project | FWA production | **new project** (fresh migrations, own Vault secrets) |
| Magic-link redirects | `fwaapp://` | new scheme, re-allowlisted in Supabase Auth |

Grep targets when re-skinning: `fwa`, `french-with-ash`, `frenchwithash`, `Ashfaaq`
in copy, `src/theme/tokens.ts`, `src/components/brand/`, and the font loading in
`app/_layout.tsx` (Lora / Work Sans → Geist / Geist Mono).

## New surfaces (beyond a pure re-skin)

The comp goes past FWA v1 scope. In build order of dependence:

1. **Four-tab shell** — home · programs · flashcards · profile replaces FWA's
   Today/History/Settings stack. The roadmap opens from the cadence strip on home;
   History moves under the roadmap.
2. **Home as instrument panel** — the day is a plan, not a list: each task
   (flashcards, homework set, listening, writing) logs a telemetry line; a cadence
   strip tracks the weekly hours promise (e.g. 7.5 / 12 hrs, "on pace · gate ≈ 3 wks").
3. **Roadmap, releases, gates** — five releases (R1–R5) with gates as first-class
   nodes: a gate is a definition-of-done checklist that unlocks from evidence, books
   a gate session into the tutor's lesson slots, leaves an artifact (e.g. the R1
   "before" recording), and is signed off by the tutor (board 38). R5 converges on
   an exam cockpit: four numbers vs the NCLC 7 thresholds, TEF booking locked until
   both mocks clear.
4. **Flashcards (SRS)** — deferred to P5 in FWA, in scope here: decks build
   themselves from chapters and from mistakes, self-graded with visible intervals,
   review is the daily habit anchor.
5. **Programs webview** — the website's programs/lesson pages, mobile-optimized,
   scoped to the student's current release (R3+ material visible but locked).
6. **Home-screen widgets** — small + medium + lock-screen: review queue, streak,
   and a streak-defence state after 20:00 (boards 18–20).
7. **Tutor side, evolved** — attention-first roster (sorted by what needs the tutor),
   student detail with the week's error heads, lesson-aware due dates ("24h before
   Friday" as the default), release-scoped library, and gate review.

The notification engine gains a fourth push type: **gate** (rare), alongside
assignment / due-soon / streak (board 32).

## Getting started (when the build begins)

1. Copy `Reference codebase (Frenchwithash)/fwa-app` to a new repo (drop
   `node_modules`, `dist`, `.git`, `google-services.json`, and FWA's EAS/Supabase
   bindings).
2. Create the new Supabase project; run the migrations; store secrets in Vault.
   Create the new EAS project. Fill `.env` from `.env.example`.
3. Do the identity swap and token re-theme above **before** feature work — the
   re-skin is a clean, reviewable first commit.
4. Then build the new surfaces in the order listed, keeping the FWA phasing
   discipline: vertical slice first, prove push + RLS on a real device, then widen.
5. `fwa-app`'s README has the working commands (`npm install`, seed scripts,
   `npx expo start`, `eas build`) — they carry over verbatim until the scripts are
   renamed.

## Reference reading order

1. `Designs/T Academy - App.dc.html` — open in a browser; every board is annotated.
2. `Reference codebase (Frenchwithash)/fwa-app/README.md` — the inherited system.
3. `PRD_French_with_Ash_Mobile_App.md` — the functional requirements (AF-1…AF-14)
   that still define the base loop.
4. The website repo's `TECHNICAL_IMPLEMENTATION_PLAN.md` and `tef.css` — the brand
   this app joins.

---

*TEF Exam Prep Academy · Montréal*
