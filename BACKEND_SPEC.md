# T Academy — Backend Technical Specification

| | |
|---|---|
| **Scope** | The complete backend for the T Academy companion app: schema, access control, derived logic, functions, jobs, sync, and integrations |
| **Baseline** | The French with Ash (FWA) Supabase backend — inherited, then extended |
| **Backend** | New dedicated **Supabase** project (Postgres + Auth + RLS + Edge Functions + Vault + pg_cron + pg_net + Storage) |
| **Consumers** | The Expo app (student + tutor trees), home-screen widgets, the tefexamprep.ca website (programs webview bridge) |
| **Status** | Draft v1 — for review via the interactive backend board (see `Designs/BACKEND_BOARD_BRIEF.md`) |
| **Date** | July 15, 2026 |

The design principle carries over from FWA verbatim: **access is enforced at the
database, not the client.** Every new table below ships with RLS on day one; every
privileged write happens under the service role, server-side; the service key never
ships in the app. The new T Academy principle layers on top: **telemetry is the
product** — every student action becomes an append-only telemetry event, and every
derived number on screen (hours banked, pace, gate ETA, streak) is computed from
that log, never stored as a mutable counter the client could drift.

---

## 1. Architecture overview

```
┌─────────────────────────┐        ┌──────────────────────────────────────────┐
│  Expo app (iOS/Android) │        │  Supabase project (new, T Academy)       │
│  student + tutor trees  │◄──────►│  Postgres (RLS on every table)           │
│  offline queue (TanStack│ anon   │  Auth (magic-link PKCE, invite-only)     │
│  Query + AsyncStorage)  │ key    │  Edge Functions (Deno)                   │
└───────────┬─────────────┘        │  Vault (service key, bridge secret)      │
            │                      │  pg_cron + pg_net (sweeps, triggers)     │
   shared app-group storage        │  Storage (gate artifacts bucket)         │
            │                      └───────┬──────────────────┬───────────────┘
┌───────────▼─────────────┐                │ Expo push API    │ HS256 bridge JWT
│  Widgets (WidgetKit /   │        ┌───────▼──────────┐  ┌────▼─────────────────┐
│  Glance) — read local   │        │  APNs / FCM      │  │ tefexamprep.ca       │
│  snapshot only (v1)     │        │  via Expo push   │  │ (Vercel + Better Auth│
└─────────────────────────┘        └──────────────────┘  │ + Neon) — programs   │
                                                         │ webview, app-bridge  │
                                                         └──────────────────────┘
```

Two backends exist on purpose and stay separate: the **website** keeps Better Auth +
Neon (marketing + lesson content + its own accounts); the **app** gets its own
Supabase project (the loop, telemetry, SRS, gates). They touch at exactly one seam —
the **programs webview bridge** (§10). Content flows one way, from the markdown
pipeline into both (§9).

---

## 2. Domain model

### 2.1 Inherited from FWA (kept as-is unless noted)

| Table | Purpose | Changes for T Academy |
|---|---|---|
| `profiles` | Account, role, segment, timezone, notification prefs, quiet hours | + journey fields move to `journeys`; + `reminders_gate` pref |
| `exercise_sets` | Content-bridge imports (Zod exercise JSON) | + `release` scoping column |
| `assignments` | Tutor assigns a set to a student | + `plan window` semantics unchanged; due default becomes lesson-aware (client-side) |
| `attempts` | Latest answer per item, verdicts `ok/accent/no/produced` | + mistake-card trigger (§7.4) |
| `daily_activity` | Per-student per-day rollup → streaks | Fed by `telemetry_events` (any kind), not only attempts |
| `push_tokens` | Expo push tokens per device | unchanged |
| `notifications_log` | Send dedup + audit | `notif_type` enum gains `gate` |

### 2.2 New tables

| Table | Cardinality | Purpose |
|---|---|---|
| `releases` | 5 static rows | R1–R5 definitions: level span, ordinal, description |
| `journeys` | 1 per student | Roadmap state: current release, start date, cadence promise, lesson slot, placement |
| `gate_templates` | 1 per release transition | The definition-of-done: criteria JSON (targets per telemetry kind, tutor-scored items, artifact requirements) |
| `gates` | 1 per student per transition | Instance + state machine: evidence auto-fill, booking, artifact, tutor review |
| `availability_slots` | tutor-maintained | Bookable lesson slots for gate sessions |
| `plan_items` | n per student per day | The daily plan: flashcards / set / listening / writing tasks |
| `telemetry_events` | append-only | Every completed action with metrics (minutes, words, cards, items) + idempotency key |
| `decks` | content + per-student | Flashcard decks: chapter decks (shared) and the personal mistakes deck |
| `cards` | n per deck | Front/back (article ships with the noun — R1 lexicon rule), source refs |
| `card_states` | 1 per student per card | SRS state: due, interval, ease, reps, lapses |
| `review_logs` | append-only | Every graded review (client-stamped UUID for offline dedup) |
| `mock_results` | n per student | TEF mock scores per skill + NCLC estimate (tutor-scored) |
| `nclc_thresholds` | static | Score → NCLC mapping per TEF skill (the cockpit's four thresholds) |

Entity relationships (crow's-foot summary):

```
profiles 1─1 journeys ─┬─ current_release ─► releases
                       └─ lesson slot (day, time, tz)
releases 1─n gate_templates (transition from→to)
profiles 1─n gates n─1 gate_templates
gates n─1 availability_slots (booked slot)   gates 1─n storage.objects (artifacts)
profiles 1─n assignments n─1 exercise_sets   assignments 1─n attempts
profiles 1─n plan_items (per local date)     plan_items 1─0..1 telemetry_events
profiles 1─n telemetry_events                telemetry_events ─► daily_activity (trigger)
decks 1─n cards; profiles 1─n card_states n─1 cards; profiles 1─n review_logs
profiles 1─n mock_results                    nclc_thresholds (lookup)
```

---

## 3. Schema (DDL sketches)

Types omitted where obvious; every table gets `created_at timestamptz default now()`.

### 3.1 Enums (new + extended)

```sql
-- extended (FWA had: new_assignment, due_soon, overdue, streak)
alter type notif_type add value 'gate';

create type release_code   as enum ('R1','R2','R3','R4','R5');
create type gate_status    as enum ('locked','in_evidence','cleared','booked','passed','deferred');
create type telemetry_kind as enum ('set_completed','cards_reviewed','listening_logged',
                                    'writing_submitted','lesson_attended','mock_completed',
                                    'gate_session');
create type plan_kind      as enum ('flashcards','set','listening','writing','custom');
create type plan_status    as enum ('pending','done','skipped');
create type card_grade     as enum ('again','hard','good','easy');
create type tef_skill      as enum ('CO','CE','EO','EE');  -- compréhension/expression, orale/écrite
create type criterion_kind as enum ('hours_banked','sets_completed','review_days',
                                    'words_written','listening_minutes','mock_nclc',
                                    'tutor_score','artifact');
```

### 3.2 Journey & releases

```sql
create table releases (
  code release_code primary key,
  ordinal int not null unique,           -- 1..5
  title text not null,                   -- 'Foundations', ...
  level_from text not null,              -- 'A0'
  level_to text not null,                -- 'A1'
  summary text
);

create table journeys (
  student_id uuid primary key references profiles(id) on delete cascade,
  current_release release_code not null default 'R1',
  started_at date not null,              -- week numbering anchor
  confirmed_at timestamptz,              -- welcome screen "Looks right — continue"
  cadence_hours_week numeric(4,1) not null default 12,
  lesson_weekday int,                    -- 0=Sun..6=Sat (e.g. 5 = Friday)
  lesson_time time,                      -- 17:00 local (profiles.timezone)
  placement_note text,                   -- 'R2 · A1 → A2, from the Jul 2 diagnostic'
  est_months int                         -- the honest timeline shown at welcome
);
```

Week number = `floor((local_today - started_at) / 7) + 1`. Day counter ("Day 62") =
`local_today - started_at + 1`. Both derived, never stored.

### 3.3 Gates

```sql
create table gate_templates (
  id uuid primary key default gen_random_uuid(),
  from_release release_code not null,
  to_release release_code not null,
  title text not null,                   -- 'Gate: R2 → R3'
  criteria jsonb not null,               -- see shape below
  unique (from_release, to_release)
);

create table gates (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references profiles(id) on delete cascade,
  template_id uuid not null references gate_templates(id),
  status gate_status not null default 'locked',
  evidence jsonb not null default '{}',  -- auto-filled per criterion (§6.4)
  cleared_at timestamptz,
  slot_id uuid references availability_slots(id),   -- booking
  artifact_path text,                    -- storage object (e.g. R1 'before' recording)
  reviewed_by uuid references profiles(id),
  review_scores jsonb,                   -- tutor-scored items at the gate session
  decision gate_status,                  -- passed | deferred (tutor sign-off, board 38)
  decided_at timestamptz,
  unique (student_id, template_id)
);

create table availability_slots (
  id uuid primary key default gen_random_uuid(),
  starts_at timestamptz not null,
  duration_min int not null default 60,
  booked_by uuid references profiles(id),           -- null = open
  unique (starts_at)
);
```

**Criteria JSON shape** (in `gate_templates.criteria`):

```json
{ "criteria": [
  {"key":"hours",   "kind":"hours_banked",     "label":"Hours banked this release",   "target":140},
  {"key":"sets",    "kind":"sets_completed",   "label":"Sets completed",              "target":24, "min_accuracy":0.8},
  {"key":"review",  "kind":"review_days",      "label":"Review days this release",    "target":40},
  {"key":"words",   "kind":"words_written",    "label":"Words written",               "target":3000},
  {"key":"mock",    "kind":"mock_nclc",        "label":"Mock ≥ NCLC 7 (both mocks)",  "skills":["EO","EE","CO","CE"], "target":7, "count":2},
  {"key":"oral",    "kind":"tutor_score",      "label":"Live mini-oral with Ash",     "scale":"pass"},
  {"key":"artifact","kind":"artifact",         "label":"Before recording uploaded"}
]}
```

**Evidence JSON shape** (in `gates.evidence`, auto-filled by `evaluate-gates`):

```json
{ "hours":   {"value":121.5, "target":140, "met":false, "as_of":"2026-07-15T04:00:00Z"},
  "artifact":{"value":true,  "met":true},
  "oral":    {"met":null}  }
```

`met: null` marks tutor-scored items — they stay pending until the gate session.

### 3.4 Plan & telemetry

```sql
create table plan_items (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references profiles(id) on delete cascade,
  plan_date date not null,               -- student-local date
  position int not null default 0,
  kind plan_kind not null,
  title text not null,                   -- 'Listening · News in Slow French'
  config jsonb not null default '{}',    -- {assignment_id} | {minutes:15,url} | {words:100,prompt} ...
  status plan_status not null default 'pending',
  completed_event_id uuid references telemetry_events(id),
  created_by uuid references profiles(id),          -- tutor, or null = system
  unique (student_id, plan_date, position)
);

create table telemetry_events (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references profiles(id) on delete cascade,
  kind telemetry_kind not null,
  occurred_at timestamptz not null default now(),
  local_date date not null,              -- computed client-side in profile tz, verified server-side
  minutes numeric(6,1) not null default 0,          -- the honest-hours currency
  metrics jsonb not null default '{}',   -- {words:120} | {cards:23,again:4} | {items:20,correct:17} ...
  assignment_id uuid references assignments(id) on delete set null,
  plan_item_id uuid references plan_items(id) on delete set null,
  release release_code,                  -- stamped from journeys.current_release at insert
  idempotency_key uuid not null unique   -- client-generated; offline queue replays are no-ops
);
create index telemetry_student_date_idx on telemetry_events (student_id, local_date);
create index telemetry_student_release_idx on telemetry_events (student_id, release);
```

**Honest-hours rule:** `minutes` is measured active time (foreground, player open,
audio playing), client-tracked, and **capped server-side** at insert by a trigger:
`set_completed` ≤ 2× the set's estimated minutes; `listening_logged` ≤ the media
duration; `cards_reviewed` ≤ 1.5 min × cards. Padding is clamped, not rejected.

### 3.5 Flashcards (SRS)

```sql
create table decks (
  id uuid primary key default gen_random_uuid(),
  student_id uuid references profiles(id) on delete cascade,  -- null = shared content deck
  source text not null check (source in ('chapter','mistakes')),
  book int, chapter int,                 -- for chapter decks
  title text not null,
  unique nulls not distinct (student_id, source, book, chapter)
);

create table cards (
  id uuid primary key default gen_random_uuid(),
  deck_id uuid not null references decks(id) on delete cascade,
  front text not null,                   -- 'the weekend' (prompt side)
  back text not null,                    -- 'le week-end' — ARTICLE SHIPS WITH THE NOUN (R1 rule)
  hint text,
  source_assignment_id uuid references assignments(id) on delete set null,
  source_item_ref text,                  -- 's2i4' — mistake cards trace to the miss
  content_hash text not null,            -- dedup on re-import / repeat mistakes
  unique (deck_id, content_hash)
);

create table card_states (
  student_id uuid not null references profiles(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  due_at timestamptz not null default now(),
  interval_days numeric(7,2) not null default 0,
  ease numeric(4,2) not null default 2.50,
  reps int not null default 0,
  lapses int not null default 0,
  suspended boolean not null default false,
  updated_at timestamptz not null default now(),
  primary key (student_id, card_id)
);

create table review_logs (
  id uuid primary key,                   -- CLIENT-generated UUID = offline idempotency
  student_id uuid not null references profiles(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  reviewed_at timestamptz not null,
  grade card_grade not null,
  interval_before numeric(7,2), interval_after numeric(7,2),
  elapsed_ms int
);
```

### 3.6 Mocks & the exam cockpit

```sql
create table mock_results (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references profiles(id) on delete cascade,
  taken_at date not null,
  skill tef_skill not null,
  raw_score int not null,
  nclc_estimate int not null,            -- tutor-entered, from the threshold table
  scored_by uuid not null references profiles(id),
  notes text
);

create table nclc_thresholds (
  skill tef_skill not null,
  nclc int not null,
  min_score int not null,                -- official TEF Canada score bands
  primary key (skill, nclc)
);
```

Cockpit logic (R5 roadmap view, board 27): for each of the four skills, take the
student's **latest full mock** score vs `nclc_thresholds` at the journey's target
NCLC. "Booking unlocks when both mocks clear" = the R5 gate's `mock_nclc` criterion
with `count: 2` — the **two most recent** full mocks must each clear all four skills.

---

## 4. Row-level security

Same doctrine as FWA: grants enable operations, policies filter rows; nothing is
granted to `anon`; `is_tutor()` (security-definer) avoids recursive lookups; `role`
is never client-updatable (column-level grant).

**Policy matrix** (S = student token, T = tutor token, SR = service role):

| Table | S select | S insert | S update | S delete | T select | T write | SR |
|---|---|---|---|---|---|---|---|
| `profiles` | own | — (invite only) | own, safe columns | — | all | — | all |
| `exercise_sets` | all | — | — | — | all | all | all |
| `assignments` | own | — | own (status/completed) | — | all | all | all |
| `attempts` | own | own | own | own | all | — | all |
| `daily_activity` | own | own | own | — | all | — | all |
| `push_tokens` | own | own | own | own | — | — | all |
| `notifications_log` | own | — | — | — | all | — | all (writer) |
| `releases`, `nclc_thresholds`, `gate_templates` | all | — | — | — | all | — | all (seeder) |
| `journeys` | own | — | — | — | all | all | all |
| `gates` | own | — | own: **booking only** (set `slot_id`, status `cleared→booked`) | — | all | all (review/decision) | all (evaluator) |
| `availability_slots` | open slots + own bookings | — | book an open slot (`booked_by = auth.uid()` when null) | — | all | all | all |
| `plan_items` | own | — | own (status, completed_event_id) | — | all | all | all |
| `telemetry_events` | own | own (with clamp trigger) | — | — | all | — | all |
| `decks` | shared + own | own (`mistakes` only) | — | — | all | all (content) | all |
| `cards` | in visible decks | own mistakes deck | — | — | all | all (content) | all |
| `card_states` | own | own | own | — | all | — | all |
| `review_logs` | own | own | — | — | all | — | all |
| `mock_results` | own | — | — | — | all | all | all |

Load-bearing details:

- **Telemetry and review logs are append-only for everyone below the service role.**
  No client update/delete grant exists — history cannot be rewritten, which is what
  makes hours, streaks, and gate evidence trustworthy.
- **Gate booking is the one student write on `gates`**, constrained by a `with check`
  that the current status is `cleared` and the target status is `booked`, and that
  the chosen slot is open (enforced transactionally in the `book_gate_slot` RPC, §8).
- **Journey advancement (`current_release`) is tutor/service-role only** — the gate
  decision (board 38) is the sole path forward.
- **Storage** (`artifacts` bucket): objects live under `{student_id}/...`; policies
  allow a student `insert/select` on their own prefix, the tutor `select` on all,
  no client deletes. Audio uploads (gate artifacts) go here.

---

## 5. The telemetry engine

The single most important design decision: **one append-only event stream feeds
every derived number.** Nothing on the home screen is a stored counter.

### 5.1 Event producers

| Action | Event kind | `minutes` | `metrics` |
|---|---|---|---|
| Set completed (player) | `set_completed` | measured active time (clamped) | `{items, correct, accent, wrong}` |
| Flashcard session ends | `cards_reviewed` | measured | `{cards, again, hard, good, easy}` |
| Listening task done | `listening_logged` | media minutes | `{url}` |
| Writing submitted | `writing_submitted` | measured | `{words}` |
| Lesson attended (tutor logs) | `lesson_attended` | 60 (slot length) | `{slot_id}` |
| Mock scored (tutor) | `mock_completed` | session length | `{mock_result_ids}` |
| Gate session (tutor) | `gate_session` | slot length | `{gate_id}` |

### 5.2 Derived rollups (views / RPCs, never tables)

- **`v_week_telemetry`** — per student per ISO week (weeks run **Monday 00:00
  student-local**): `hours = sum(minutes)/60`, `words`, `listening_min`, `cards`.
  Powers the cadence strip ("7.5 / 12 hrs").
- **`daily_activity`** — kept from FWA for streaks, but now written by an
  `AFTER INSERT` trigger on `telemetry_events` (any kind counts), not on attempts.
  FWA's attempts trigger is dropped; item-level activity arrives via `set_completed`.
- **Pace** — `on_pace` iff `hours_this_week ≥ cadence_hours_week × elapsed_week_fraction − 1.0`
  (one grace hour). Elapsed fraction is computed in the student's tz.
- **Gate ETA** ("gate ≈ 3 wks") — per auto criterion:
  `remaining / trailing_4wk_weekly_rate`, take the **max** across criteria, round up
  to whole weeks; `∞` renders as "—" if the trailing rate is 0.
- **Streak** — unchanged FWA algorithm: consecutive student-local days with a
  `daily_activity` row; at-risk = active streak ≥ 1 and nothing logged today.

### 5.3 Release stamping

Each event is stamped with `release` (from `journeys.current_release`) by a `BEFORE
INSERT` trigger. Gate criteria of kind `hours_banked` / `sets_completed` / etc. are
computed **within the current release's events** — hours banked in R1 don't count
toward the R2→R3 gate.

---

## 6. The gate state machine

```
             evidence auto-criteria all met            student books a slot
 locked ──► in_evidence ────────────────► cleared ────────────────► booked
 (future     (release is                  (CTA live,                 │ gate session happens
  releases)   active)                      'gate' push)              ▼ tutor signs (board 38)
                                                          passed ──► journeys.current_release
                                                             │        advances; next gate
                                                          deferred    flips to in_evidence
                                                          (stays booked-able again)
```

- **`evaluate-gates`** (Edge Function, §8) recomputes `evidence` for every
  `in_evidence` gate: nightly by cron, plus on-demand after each `set_completed`
  telemetry insert (cheap: single-student scope, via pg_net trigger).
- Transition `in_evidence → cleared` happens **only inside `evaluate-gates`**
  (service role), fires the `gate` push once (dedup key `gate:<gate_id>:cleared`),
  and stamps `cleared_at`.
- **Booking** (`book_gate_slot` RPC): atomically claims an open `availability_slot`
  and moves the gate to `booked`. Double-booking is impossible: the slot update is
  `where booked_by is null`, checked rows = 1 or the whole RPC raises.
- **Tutor review** (board 38): telemetry auto-fills the evidence panel; the tutor
  scores the live items (`review_scores`), sets `decision` = `passed`/`deferred`.
  `passed` advances `journeys.current_release`, activates the next gate
  (`locked → in_evidence`), and logs a `gate_session` telemetry event.
- **R5's gate is the exam booking** — its criteria are the cockpit thresholds; its
  "booking" is the real TEF sitting (tracked, not sold in-app).

---

## 7. Flashcards: scheduling and deck-building

### 7.1 Algorithm decision

**v1 ships SM-2** (the Anki-family classic) as a **pure, shared TypeScript module**
(`src/lib/srs.ts`) — same pattern as the FWA `judge`: no I/O, unit-tested with fixed
clocks, imported by both the app and any server code. FSRS is the named successor
(P5+), a drop-in behind the same `(state, grade, now) → state` signature.

### 7.2 SM-2 parameters (exact)

```
state: {interval_days, ease, reps, lapses}   grade ∈ {again, hard, good, easy}

again: lapses+1, reps=0, interval=0 (relearn: due in 10 min), ease=max(1.30, ease−0.20)
hard:  interval = max(1, interval×1.2),        ease = max(1.30, ease−0.15)
good:  reps 0→ interval=1; reps 1→ interval=3; else interval = round(interval×ease)
easy:  interval = round(max(1,interval)×ease×1.3), ease = ease+0.15
→ due_at = now + interval_days (10 min for 'again'); reps+1 on hard/good/easy
```

Daily caps (client-enforced, visible in UI): 10 new cards, 60 reviews. **Intervals
stay visible on the grade buttons** (board 16) — "the system reads honest, not
magical."

### 7.3 Offline-first grading

Reviews are graded **on-device** (the shared module), queued as `review_logs` rows
with client-generated UUID primary keys, and synced with the matching `card_states`
upsert. Replays are no-ops (PK conflict → ignore). `card_states` conflicts resolve
last-write-wins by `reviewed_at` — acceptable because a student reviews on one
device at a time and the data is theirs alone.

### 7.4 Deck building

- **Chapter decks** (shared, `student_id = null`): built by the content bridge from
  each chapter's vocabulary — **the article ships with the noun** (`le week-end`,
  never `week-end`), enforced at export by the bridge's card formatter.
- **Mistakes deck** (per student): an `AFTER INSERT OR UPDATE` trigger on `attempts`
  where `verdict in ('no','accent')` upserts a card into the student's mistakes deck
  (front = the prompt, back = the canonical answer, hash-deduped so a repeated miss
  strengthens the existing card's queue position rather than duplicating).
- A `card_states` row is lazily created at first sight (client) or by the mistake
  trigger (server) with `due_at = now()`.

---

## 8. Server surface: Edge Functions, RPCs, triggers, cron

### 8.1 Edge Functions (Deno, service role via Vault)

| Function | Trigger | Does |
|---|---|---|
| `send-new-assignment` | pg_net from `AFTER INSERT` on assignments (inherited) | Immediate push: "Ash sent you a new set: *title*. Due *date*." Dedup via notifications_log |
| `send-reminders` | pg_cron hourly (inherited) | The sweep: due-soon (24h lead) / overdue / streak (evening ≥18:00) — quiet hours 21:00–08:00, daily cap 3, priority due>overdue>streak, dedup keys as in FWA `_shared/reminders.ts` |
| `evaluate-gates` | pg_cron nightly 04:00 UTC + pg_net after `set_completed` inserts | Recompute evidence for `in_evidence` gates; flip to `cleared`; fire the `gate` push (rare, dedup `gate:<id>:cleared`) |
| `delete-account` | client (Settings) | Inherited; cascade now covers journeys, gates, telemetry, card data, mocks — verified by test |
| `mint-web-session` | client (opening Programs tab) | Signs a 5-minute HS256 JWT for the website bridge (§10) |

### 8.2 Postgres RPCs (security definer, called by the app)

| RPC | Caller | Does |
|---|---|---|
| `book_gate_slot(gate_id, slot_id)` | student | Atomic slot claim + `cleared → booked` (§6) |
| `get_daily_plan(date)` | student | Returns plan_items for the local date; **materializes on first read**: the standing flashcards task + tasks derived from open assignments and the student's plan template |
| `week_summary(week_start)` | student/tutor | The cadence strip numbers from `v_week_telemetry` + pace verdict |
| `roster_attention()` | tutor | Roster sorted by attention score (§8.4) |
| `student_detail(student_id)` | tutor | Journey + week telemetry + error heads + gate evidence in one round trip |

### 8.3 Triggers & cron (complete inventory)

| Object | On | Purpose |
|---|---|---|
| `assignments_notify_new` | AFTER INSERT assignments | pg_net → `send-new-assignment` (Vault key; no-ops if secret absent) |
| `telemetry_stamp_release` | BEFORE INSERT telemetry_events | Stamp `release`, verify `local_date` against profile tz (±1 day tolerance), clamp `minutes` (§3.4) |
| `telemetry_rollup_activity` | AFTER INSERT telemetry_events | Upsert `daily_activity` for the student-local date |
| `telemetry_gate_check` | AFTER INSERT telemetry_events (`set_completed` only) | pg_net → `evaluate-gates` scoped to the student |
| `attempts_mistake_card` | AFTER INSERT/UPDATE attempts | Verdict `no`/`accent` → upsert mistakes-deck card (§7.4) |
| `touch_updated_at` | exercise_sets, attempts, card_states | Inherited |
| cron `send-reminders-hourly` | `0 * * * *` | The reminder sweep |
| cron `evaluate-gates-nightly` | `0 4 * * *` | Full evidence recompute (catches tutor-side edits, clock drift) |

### 8.4 Derived views

- `v_week_telemetry` (§5.2) · `v_review_queue` (due card_states joined to cards,
  ordered by `due_at`, capped) · `v_error_heads` (last-7-day attempts grouped by
  set chapter + item kind, wrong/accent counts — board 35's "week's error heads")
- `roster_attention` sort key (board 34), descending:
  `overdue_assignments×3 + hours_behind_pace×2 + streak_at_risk×2 + gate_cleared_unbooked×4 + days_since_active`.

---

## 9. Content pipeline

Inherited and extended:

```
markdown workbooks ──content-bridge──► exercise JSON (Zod ExerciseContent)
                                   └─► chapter vocab → card JSON (front/back, article rule)
seed scripts (service role, local) ──► exercise_sets (hash-skip unchanged)
                                   └─► decks + cards (hash-skip unchanged)
```

- `exercise_sets` gains `release release_code` — the library is **release-scoped**
  (board 37): R3+ sets are *visible but locked* for a student below R3. Enforcement
  is client-side display + a tutor-side warning; RLS intentionally does not hide
  them (the tutor may assign ahead deliberately).
- The `item_ref` convention (`s2i4`, `s1f0`) and the accent-aware judge verdicts
  (`ok/accent/no/produced`) are unchanged — the mistake-card trigger depends on them.

---

## 10. Programs webview bridge (app ↔ website)

The Programs tab renders **tefexamprep.ca**, mobile-optimized, release-scoped
(boards 21–23). The website keeps its own auth (Better Auth/Neon); the app must not
force a second login, and **no token ever rides a URL**.

**Handshake (decision D-3):**

1. App calls Edge Function `mint-web-session` (authenticated with the Supabase JWT).
   It signs `{sub: user_id, email, release, aud: 'tef-web', exp: now+5m}` with
   `APP_BRIDGE_SECRET` (HS256) — the secret lives in Supabase Vault **and** the
   Vercel project env; it is the only shared credential between the two backends.
2. The webview's **initial request** is a POST to
   `https://tefexamprep.ca/api/app-bridge` with the JWT in the `Authorization`
   header (Expo `WebView` supports headers on the first request; the token is never
   in a query string).
3. The website verifies the JWT, upserts/links a website account by email, creates a
   Better Auth session **cookie**, and 302s to `/programs?embedded=1` (`embedded`
   hides site chrome — a display flag, not a credential).
4. The cookie carries the session for all subsequent webview navigation. The
   website's middleware reads the session's `release` claim and locks R3+ lesson
   pages (visible, gated — matching the library rule).

Token replay is bounded by the 5-minute `exp` + single-use `jti` cache on the
website side. Sign-out in the app does not need to propagate (the cookie dies with
the webview session; `exp` bounds any residue).

---

## 11. Notifications (complete engine)

Four push types (board 32):

| Type | Path | Timing | Dedup key |
|---|---|---|---|
| `new_assignment` | trigger → Edge Function | immediate | `new_assignment:<assignment_id>` |
| `due_soon` | hourly sweep | ≤ 24h before `due_at` | `due_soon:<assignment_id>` |
| `overdue` | hourly sweep | past due, gentle | `overdue:<assignment_id>` |
| `streak` | hourly sweep | evening (≥18:00 local), streak at risk, nothing logged | `streak:<local_date>` |
| `gate` | `evaluate-gates` | on `cleared` (rare) | `gate:<gate_id>:cleared` |

Constraints (all sends, inherited engine): student-timezone aware; quiet hours
21:00–08:00 default (per-profile override); daily cap 3 with priority
`new_assignment` = `gate` > `due_soon` > `overdue` > `streak`; per-type opt-outs on
the profile (`reminders_due_soon/overdue/streak` + new `reminders_gate`); every send
logged to `notifications_log`. The eligibility engine stays a **pure module**
(`_shared/reminders.ts`), extended with the `gate` type, unit-tested across
timezones with fixed clocks.

**Due-date default is lesson-aware** (board 36): the assign sheet defaults `due_at`
to `next lesson_weekday/lesson_time − 24h` — computed client-side from `journeys`;
the reminder engine needs no change (it just sees `due_at`).

---

## 12. Offline & sync model

| Data | Strategy |
|---|---|
| Assigned sets, decks, plan | Cached on fetch (TanStack Query persist → AsyncStorage) |
| Attempts | Upsert queue, `unique (assignment_id, item_ref)`, last-write-wins |
| Telemetry events | Queue with client `idempotency_key` — replay-safe |
| Review logs | Client-UUID PK — replay-safe; card_states LWW by `reviewed_at` |
| Completions | Assignment status update queued; server accepts monotonic transitions only (`assigned→in_progress→completed`) |
| Widgets | v1 reads a **local snapshot** (shared app-group storage) written by the app on foreground/background transitions — no network, no token in the widget process |

The clamp trigger (§3.4) and append-only RLS mean a hostile or drifted client can
at worst under-report — it cannot inflate hours or rewrite history.

---

## 13. Security model & MASVS mapping

Everything FWA proved carries forward and now covers the new tables:

- Invite-only auth (magic link + PKCE); reviewer password path retained.
- RLS on **every** table (matrix §4); cross-student isolation proven by the
  inherited `rls.integration` test, extended to telemetry/cards/gates.
- No privilege escalation: `role` and `journeys.current_release` unreachable from
  client tokens (column grants + tutor/SR-only policies).
- Service key only in Vault + Edge Functions; bridge secret only in Vault + Vercel.
- Tokens in expo-secure-store; no personal data in URLs (bridge uses POST header).
- Append-only telemetry/review history; server-side clamps on self-reported time.
- Account deletion cascades across all 20 tables + storage prefix (verified test).

MASVS anchors: V1 (architecture: this doc), V2 (data storage: secure store, no
plaintext), V4 (auth: invite-only, PKCE, short-lived bridge JWT), V6 (network:
HTTPS-only, header-borne credentials).

---

## 14. Migration plan (ordered)

| # | Migration | Contents |
|---|---|---|
| 01 | `init` | FWA base schema + RLS, with `notif_type` including `gate` and `exercise_sets.release` from day one |
| 02 | `journeys_releases` | `releases` (seeded), `journeys` + policies |
| 03 | `telemetry` | `telemetry_events`, clamp/stamp/rollup triggers, `daily_activity` rewire (drop attempts trigger) |
| 04 | `plan` | `plan_items` + `get_daily_plan` RPC |
| 05 | `flashcards` | `decks`, `cards`, `card_states`, `review_logs`, mistake trigger |
| 06 | `gates` | `gate_templates` (seeded), `gates`, `availability_slots`, `book_gate_slot`, storage bucket + policies |
| 07 | `mocks` | `mock_results`, `nclc_thresholds` (seeded) |
| 08 | `notify` | assignment trigger, gate-check trigger, `send-reminders` + `evaluate-gates` cron |
| 09 | `views` | `v_week_telemetry`, `v_review_queue`, `v_error_heads`, `roster_attention`, `student_detail` |

Seed order: releases → gate_templates → nclc_thresholds → content (sets + chapter
decks via bridge) → demo journey. All seeds idempotent (hash-skip / upsert).

---

## 15. Decisions and open questions

Made and held (revisit only with cause):

| # | Decision |
|---|---|
| D-1 | **Two backends, one seam.** The website keeps Better Auth/Neon; the app gets its own Supabase project; they meet only at the bridge JWT (§10) |
| D-2 | **Telemetry is append-only and canonical.** No mutable counters; every screen number is derived |
| D-3 | **Bridge = short-lived HS256 JWT over POST header → website cookie.** No tokens in URLs, no second login |
| D-4 | **SM-2 in v1**, as a pure shared module; FSRS later behind the same signature |
| D-5 | **Client-side SRS grading with idempotent sync** — offline-first beats server-authoritative for single-user data |
| D-6 | **Gates advance only by tutor decision**; evidence unlocks the CTA, never the release |
| D-7 | **Widgets read a local snapshot in v1** — no widget-process network or tokens |
| D-8 | **Release-scoping of the library is advisory** (visible-but-locked), enforced socially by the tutor, not by RLS |

Open (need Ash's call before build):

| # | Question | Options |
|---|---|---|
| O-1 | Gate criteria numbers per release (hours, sets, words targets) | Draft from the 12 hrs/wk × est_months math; tutor tunes in `gate_templates` seed |
| O-2 | Website account linking on first bridge: auto-create, or require an existing website account? | Auto-create by email (recommended) vs invite-scoped |
| O-3 | Does `lesson_attended` telemetry come from the tutor's tap or calendar integration? | Manual tutor action in v1 (recommended) |
| O-4 | Mock cadence in R5 (how many mocks scheduled, spacing) | Product call; schema already supports any |
| O-5 | Plan templates: fixed weekly template per student vs tutor hand-planning each week | Template + overrides (recommended) |

---

*T Academy · TEF Exam Prep Academy · Montréal*
