---
name: roster
description: Manage a team's roster — the set of players that make up a team, formed out of a try-out, capped by max_size and gated by age_group + gender eligibility. Create or open a roster for a team, add or remove players (members), set the roster from a try-out's outcomes, enforce capacity + eligibility, lock it, and view it. Adding or removing a player is a three-surface write — the roster, the member's teams[], and the team's roster_id all move together or none do. Triggered when the user wants to build, change, set, lock, or read a team's roster — e.g. "/roster open team-2014-boys-red", "/roster add member mbr-0182 to team-2014-boys-red", "set the U10 girls roster from tryout try-0007", "lock the roster for team-2014-boys-red", "/roster view team-2014-boys-red".
---

# /roster — manage a team's roster

A **roster** is the set of players that make up a team. It is formed
out of a **try-out**, capped by a `max_size`, and gated so every
player on it matches the team's `age_group` and `gender`. This skill
owns the roster record end-to-end: open it, fill it, enforce its
limits, lock it, read it — and keep the three surfaces a roster touches
(the roster, the member, the team) in agreement at all times.

This is club-operations data, not coaching. `/roster` decides **who is
on the team**. It does not decide playing time, formations, or
development — those belong to the people in their role-domains
(`rasa.domain.soccer.coach`, `rasa.domain.soccer.player`, and the live
`rasa.domain.soccer.goalkeeping`). Those references are **soft**: the
roster is complete and correct whether or not a role-domain is present.

The contract for the whole platform lives in
`.claude/club-ops-rules.md`. Roster records live at
`state/rosters/<id>.md`, created from `state/rosters/_template.md`.

## Behavior contract

- **A roster belongs to exactly one team, and a team has exactly one
  roster.** The link is bidirectional: `Roster.team` names the team id;
  `Team.roster_id` names the roster id. These two fields are a matched
  pair — if one points somewhere, the other must point back. A team
  with `roster_id` set but no roster record (or a roster whose `team`
  names a team that doesn't point back) is **drift**; this skill
  refuses to create it and flags it if it finds it.

- **The roster is the source of truth for membership; the member and
  the team mirror it.** `Roster.players[]` is the authoritative list of
  member ids on the team. Each of those members carries the team id in
  its own `teams[]`, and the team carries the roster id in `roster_id`.
  The roster leads; the other two surfaces follow.

- **Every add/remove is a three-surface write — all or none.** Adding
  a player touches THREE records: the roster (`players[]`), the member
  (`teams[]`), and — when the roster is first bound to its team — the
  team (`roster_id`). Removing reverses the first two. **A change lands
  in every surface it touches or in none of them.** Never write the
  roster's `players[]` and leave the member's `teams[]` stale. If you
  cannot complete all the writes (a record is missing, malformed, or
  locked), make NO writes and report what blocked you. This is the
  load-bearing invariant of the skill — repos-style integrity, the same
  rule the whole platform runs on.

- **Capacity is hard.** `max_size` is a ceiling, not a suggestion. An
  add that would make `len(players[])` exceed `max_size` is refused.
  No silent over-fill, no "just this one." If the team genuinely needs
  more spots, the user raises `max_size` first (an explicit edit), then
  adds.

- **Eligibility is hard.** A player goes on a roster only if the
  member's `age_group` AND `gender` both match the team's `age_group`
  and `gender`. A U10 girls team takes only members whose
  `age_group: U10` and `gender: girls`. A mismatch on either dimension
  is refused with the specific mismatch named. (The `age_groups` and
  `genders` vocabularies are defined in `state/club.md` — eligibility
  checks against those canonical values, not free text.)

- **Status gates what's allowed.** A roster's `status` is one of
  `forming | set | locked`:
  - `forming` — open for edits. Add, remove, set-from-tryout all work.
  - `set` — the intended squad is assembled and confirmed, but not yet
    frozen. Edits still work; `set` is a signal, not a lock.
  - `locked` — frozen. No add, no remove, no set-from-tryout. A locked
    roster is read-only until a user explicitly unlocks it (back to
    `set`), which is a deliberate, logged action, not a side effect.

- **Only players go on a roster.** A member with `type: parent` is
  never added — `players[]` holds member ids whose `type: player`.
  Reject a parent id with that reason.

- **The skill owns the machine fields; humans own the prose.** This
  skill writes the YAML frontmatter and the orchestrator-owned sections
  of the record. The human-authored zones (notes, context) are left
  untouched on every edit.

## Entity fields this skill reads and writes

The canonical field sets — identical across every club-ops skill and
template. Do not invent or rename fields.

- **Roster** — `id`, `team` (a Team id), `players[]` (Member ids,
  all `type: player`), `from_tryout` (a Tryout id or empty), `max_size`
  (integer), `status` (`forming | set | locked`).
- **Team** — `id`, `name`, `age_group`, `gender`, `season`,
  `coaches[]` (Staff ids), `roster_id` (this roster's id),
  `home_complex` (**phase 2** — a Complex id; may be empty, degrade
  gracefully), `leagues[]` (**phase 2**), `tournaments[]` (**phase 2**),
  `status`.
- **Member** — `id`, `name`, `type` (`player | parent`), `age_group`
  (players), `gender` (players), `parents[]` (players) / `players[]`
  (parents), `contact`, `status` (`active | inactive | prospect`),
  `registrations[]`, `teams[]` (players — holds the team ids this
  player is rostered on).
- **Tryout** (read-only here, for set-from-tryout) — `id`, `age_group`,
  `gender`, `date`, `location`, `evaluators[]`, `candidates[]`,
  `outcomes` (the offer/cut decisions this skill reads to assign
  players), `status` (`scheduled | complete`).

## Operations

### 1. Open a roster for a team — `/roster open <team-id>`

Create the roster record and bind it to its team.

1. Read the team record at `state/teams/<team-id>.md`. If absent, stop:
   you cannot roster a team that doesn't exist (point the user at
   `/team` to create it first).
2. If `Team.roster_id` is already set, this team already has a roster —
   don't create a second one. Open the existing one instead (jump to
   `/roster view`) and say so.
3. Allocate a roster `id` (e.g. `ros-<short>`). Create
   `state/rosters/<id>.md` from `state/rosters/_template.md` with:
   `team: <team-id>`, `players: []`, `from_tryout:` empty,
   `max_size:` the value the user gives (or the team's default if the
   club defines one in `state/club.md`), `status: forming`.
4. **Two-surface bind:** write `Team.roster_id = <id>` on the team
   record. The roster's `team` and the team's `roster_id` now point at
   each other. If you cannot write the team record, delete the
   half-created roster and report — no orphan rosters.
5. Confirm: roster `<id>` opened for `<team.name>` (`<age_group>`
   `<gender>`), `max_size: <n>`, 0 / `<n>` players, `status: forming`.

### 2. Add a player — `/roster add <member-id> to <team-id>`

The core three-surface write. Run the gates in order; stop at the first
failure and make NO writes.

1. **Resolve the roster.** Read the team, follow `roster_id` to the
   roster record. No `roster_id` → tell the user to `/roster open` first.
2. **Status gate.** Roster `locked` → refuse (read-only). `forming` or
   `set` → proceed.
3. **Member gate.** Read `state/members/<member-id>.md`. Absent →
   refuse. `type: parent` → refuse (parents don't roster). Already in
   `players[]` → no-op, say so (idempotent).
4. **Eligibility gate.** `Member.age_group == Team.age_group` AND
   `Member.gender == Team.gender`. Mismatch on either → refuse, naming
   which dimension and both values (e.g. "member is U11 / boys; team is
   U10 / boys — age_group mismatch").
5. **Capacity gate.** `len(players[]) + 1 <= max_size`. Over →
   refuse, showing `<current>/<max_size>`. Tell the user to raise
   `max_size` if the spot is real.
6. **Commit all surfaces (all-or-none):**
   - Roster: append `<member-id>` to `players[]`.
   - Member: append `<team-id>` to that member's `teams[]`.
   - Team: ensure `roster_id` points at this roster (already true if
     opened via op 1; set it if somehow unset — never leave it stale).
   Write all the records. If any single write fails, **revert the ones
   that succeeded** and report — the run leaves all three surfaces as it
   found them or fully updated, never half.
7. Confirm: `<member.name>` added to `<team.name>` — now
   `<count>/<max_size>`, member's `teams[]` updated.

### 3. Remove a player — `/roster remove <member-id> from <team-id>`

The reverse two-surface write.

1. Resolve the roster (as in op 2). **Status gate:** `locked` → refuse.
2. If `<member-id>` is not in `players[]` → no-op, say so.
3. **Commit both surfaces (all-or-none):**
   - Roster: drop `<member-id>` from `players[]`.
   - Member: drop `<team-id>` from that member's `teams[]`.
   The team's `roster_id` does NOT change on a remove — the roster
   still exists, just with one fewer player. Both writes land or
   neither does.
4. Confirm: `<member.name>` removed from `<team.name>` — now
   `<count>/<max_size>`, member's `teams[]` updated.

### 4. Set the roster from a try-out — `/roster set <team-id> from <tryout-id>`

Turn a completed try-out's outcomes into roster membership in one move.
This is how rosters are normally born; manual add (op 2) is for
post-tryout adjustments.

1. **Resolve both ends.** Read the team → its roster (open one via
   op 1 if none exists). Read `state/tryouts/<tryout-id>.md`.
2. **Try-out gates.** Tryout `status` must be `complete` (you don't
   roster from a tryout that hasn't run). Tryout `age_group` and
   `gender` must match the team's — a U10-girls roster sets only from a
   U10-girls try-out; mismatch → refuse, naming the dimension.
3. **Roster status gate.** `locked` → refuse. Otherwise proceed.
4. **Read the outcomes.** From the tryout's `outcomes`, take the
   members marked offered/accepted (the roster assignments) — ignore
   cuts and pending. These are the candidate ids to roster.
5. **Re-run the per-player gates** (member exists, `type: player`,
   eligibility, and the running capacity count) for each assigned
   member, in order. Any single assignment that fails a gate is
   **skipped and reported** — set-from-tryout does not silently drop a
   player or blow past `max_size`. The ones that pass are added via the
   op-2 three-surface write.
6. Record provenance: set `Roster.from_tryout = <tryout-id>`.
7. Optionally advance `status` from `forming` to `set` if the user
   asked to (set-from-tryout assembles the intended squad). Don't lock
   here — locking is its own deliberate op.
8. Confirm: a summary — `<k>` players added from `<tryout-id>`,
   `<m>` skipped (with reasons), roster now `<count>/<max_size>`,
   `from_tryout` recorded.

### 5. Lock / unlock the roster — `/roster lock <team-id>` · `/roster unlock <team-id>`

1. **Lock:** resolve the roster; set `status: locked`. From this point
   add / remove / set-from-tryout all refuse. A team usually locks its
   roster at the start of a season once the squad is final.
2. **Unlock:** set `status` back to `set` (not `forming` — unlocking
   means "I need to make a correction to a finalized roster," which
   starts from a confirmed squad). Unlock is deliberate and explicit —
   the user asked for it by name; never unlock as a side effect of
   another op.
3. Confirm the new status either way.

### 6. View a roster — `/roster view <team-id>`

Read-only. Resolve the roster from the team and render:

- Header: team name, `age_group` / `gender`, season, `status`,
  `<count>/<max_size>`, `from_tryout` (or "manual / none").
- The player list: for each id in `players[]`, the member's name and
  `status` (`active | inactive | prospect`). If a role-domain is
  present you MAY enrich a row (e.g. a goalkeeping note from
  `rasa.domain.soccer.goalkeeping`) — but absence is fine; the view is
  complete from club-ops data alone.
- **Integrity check (always):** verify the three surfaces agree —
  `Team.roster_id` points back at this roster, and every member in
  `players[]` carries `<team-id>` in their `teams[]`. Report any
  mismatch as drift with the exact records involved. A `/roster view`
  that hides drift is worse than no view.

## When NOT to use this skill

- **Creating or editing the team itself** (name, season, coaches,
  `age_group`, `gender`, `home_complex`) → use `/team`. `/roster` reads
  the team and writes only its `roster_id`; everything else about the
  team is `/team`'s.
- **Creating or editing a member** (a player's `age_group`, `gender`,
  `contact`, registrations) → use `/member`. `/roster` reads members
  and writes only their `teams[]`; it never edits a player's own
  attributes. (If a player's `age_group`/`gender` is wrong, fix it in
  `/member` first, then roster.)
- **Running the try-out itself** (scheduling, evaluators, candidates,
  recording outcomes) → use `/tryout`. `/roster` only *reads* a
  completed try-out's outcomes in op 4; it never sets them.
- **Coaching decisions** — lineups, positions, minutes, development.
  Those are the people's role-domains, not club-ops data. The roster
  says who is on the team; it is silent on what they do once they are.
- **Phase-2 surfaces** — leagues, tournaments, fields, complexes,
  schedules, registrations. `/roster` references a team's
  `home_complex` / `leagues[]` / `tournaments[]` only as labels and
  degrades gracefully when absent; it does not manage them.

## What "done" looks like

- `state/rosters/<id>.md` exists, bound to its team, with `players[]`,
  `from_tryout`, `max_size`, and `status` all set.
- **The three surfaces agree, with no exception:** `Roster.team` and
  `Team.roster_id` point at each other; every id in `Roster.players[]`
  carries the team id in its own `teams[]`. No half-writes anywhere —
  the run left every surface it touched fully updated, or it touched
  nothing and reported why.
- `max_size` was never exceeded and no ineligible (wrong
  `age_group`/`gender`, or `type: parent`) player is on the roster.
- A locked roster reads back as `locked` and rejected every mutation
  attempted against it.
- One clear confirmation in chat: what changed, the new
  `<count>/<max_size>`, and any skipped players with reasons.
