---
name: tryout
description: Run a TRY-OUT event end-to-end for the club — schedule it (age_group, gender, date, location, evaluators from staff), register candidates from members, record evaluations and outcomes, produce roster assignments, hand off to /roster, and close. Writes state/tryouts/<id>.md. Triggered when the user wants to set up, run, evaluate, or close a tryout — e.g. "/tryout", "schedule a U12 boys tryout", "register candidates for the spring tryout", "record evaluations", "who made the team", "close out the tryout".
---

# /tryout — Run a try-out event end-to-end

A try-out is the club's **selection pipeline**: a single dated event for one
`age_group` + `gender`, staffed by evaluators (staff), open to candidates
(members), that produces **roster assignments** which feed `/roster`. This
skill owns the Tryout record from `scheduled` to `complete` and keeps every
related entity consistent on the way.

The orchestrator owns the *event* — the who/when/where and the outcomes. It
does NOT do the evaluating itself. A coach's judgment of a player lives in the
coach's role-domain (`rasa.domain.soccer.coach`, soft); the live
`rasa.domain.soccer.goalkeeping` toolkit is what a GK evaluator reaches for.
The Tryout record captures *that* an outcome was recorded and *what* it was —
not how the evaluator arrived at it.

Reads/writes `state/tryouts/<id>.md` (from `state/tryouts/_template.md`).
Pairs with `.claude/club-ops-rules.md` (the canonical entity model + integrity
rules) and hands off to `/roster`. Soft-references `/member`, `/staff`, and
`/team`.

## Behavior contract

### The entity (canonical — do not drift)

A **Tryout** record carries exactly these fields (per `.claude/club-ops-rules.md`):

- `id` — slug, e.g. `tryout-u12-boys-2026-spring`
- `age_group` — one of the club's `age_groups` dimension (e.g. `U6`…`U19`, `Adult`)
- `gender` — one of `boys` | `girls` | `coed`
- `date` — ISO `YYYY-MM-DD` (single-day event; multi-session is phase 2)
- `location` — a complex or field reference (**phase 2 entity** — degrade gracefully; see below)
- `evaluators[]` — staff ids (the people doing the assessing)
- `candidates[]` — member ids (players trying out), each matched on `age_group` + `gender`
- `outcomes` — per-candidate result that becomes a roster assignment (see Operation 3)
- `status` — `scheduled` | `complete`

Never invent fields. If a club wants something the model doesn't carry
(e.g. a station rotation), record it in the human-authored notes zone of the
record, not as a new frontmatter key — and flag it for a possible rules update.

### Discover before you write

On invocation, read in this order:

1. `state/tryouts/<id>.md` if an id is named or inferable — resume, don't restart.
2. `state/tryouts/` index/listing — find an in-flight `scheduled` tryout matching the `age_group` + `gender` the user mentioned before creating a new one.
3. `state/club.md` — the `age_groups`, `genders`, and (phase 2) complexes/fields dimensions. **Validate `age_group` and `gender` against this** — don't accept a value the club hasn't defined. If the dimension is missing, say so and point the user at `/member` or club setup rather than guessing.
4. `state/staff/` — to resolve evaluator ids (must be `status: active`, `role: coach|director`).
5. `state/members/` — to resolve candidate ids (players matching the tryout's `age_group` + `gender`).

Surface what you found before asking. "Found `tryout-u12-boys-2026-spring`
(scheduled, 3 evaluators, 18 candidates, 0 outcomes). Recording evaluations?
[yes / something else]"

### Integrity is non-negotiable

A try-out touches multiple entities. Per the `club-ops-rules.md` integrity
convention (repos-style: a change lands in *all* the surfaces it touches or
*none*):

- An evaluator id MUST resolve to a real `state/staff/<id>.md` with `status: active`. A candidate id MUST resolve to a real `state/members/<id>.md` of `type: player`.
- A candidate's `age_group` + `gender` MUST match the tryout's. Mismatch is an error, not a warning — refuse and explain. (A U10 girl is not a candidate for a U12 boys tryout.)
- When a candidate is registered, the Tryout's `candidates[]` is the source of truth for *this event*. Do not mutate the member's `teams[]` here — team placement happens at `/roster`, not at tryout. The tryout produces *assignments*; `/roster` commits them.
- Outcomes only exist for candidates already in `candidates[]`. You cannot record an outcome for someone who never registered.

### Never auto-commit

Write the record, `git add` it so the diff is reviewable, tell the user what
changed and where. The human commits. (Same convention as every club-ops
skill.)

## Operations

The skill runs the four phases of a tryout. They're ordered but resumable —
pick up wherever the record is.

### 1. Schedule a tryout

Create a new Tryout record. Required to proceed: `age_group`, `gender`, `date`.

1. Ask for / confirm `age_group` and `gender`; validate both against `state/club.md` dimensions. Reject undefined values.
2. Ask for `date` (ISO `YYYY-MM-DD`). One date per tryout.
3. **Location (phase 2 — graceful).** Ask "Where? (complex or field)". Complexes/Fields are phase-2 entities — there is no `state/complexes/` skill yet. So:
   - If the user names a place, store it as a free-text `location` string and note "(phase 2: not yet a linked Complex/Field record)".
   - If they don't, leave `location` as `TBD` and move on. **Never block scheduling on location.**
4. **Evaluators from staff.** Ask which staff evaluate. Resolve each name to a `state/staff/<id>.md` (active coach/director). Put their ids in `evaluators[]`. If a named evaluator isn't in staff, offer to route to `/staff` to create them — don't silently fabricate an id.
5. Derive the `id` slug: `tryout-<age_group-lowercased>-<gender>-<date-or-season>` (e.g. `tryout-u12-boys-2026-spring`). Confirm with the user; ids are permanent.
6. Write `state/tryouts/<id>.md` from `_template.md` with `status: scheduled`, empty `candidates[]`, empty `outcomes`. `git add` it.

### 2. Register candidates

Add members to `candidates[]`. The tryout must already exist (`status: scheduled`).

1. Identify the candidate pool: read `state/members/` for `type: player`, `status: active|prospect`, matching the tryout's `age_group` + `gender`. Offer the matching list — "12 eligible U12 boys; register all, or a subset?"
2. For each member the user selects:
   - Confirm the member's `age_group` + `gender` match the tryout. Mismatch → refuse that one, explain, continue with the rest.
   - Append the member id to `candidates[]` (idempotent — no duplicates).
3. A walk-up candidate who isn't a member yet → route to `/member` to create the member record first (as a `prospect`), THEN register. The tryout references member ids; it does not hold ad-hoc names.
4. Re-write the record, `git add`, report the new candidate count.

### 3. Record evaluations / outcomes

Capture results per candidate. This is where evaluators' judgments land as data.

1. For each candidate in `candidates[]`, record an entry in `outcomes`. Each outcome carries:
   - `candidate` — member id (must be in `candidates[]`)
   - `result` — the disposition that becomes a roster assignment: `assigned` | `waitlist` | `cut` | `pending`
   - `assigned_team` — a `state/teams/<id>.md` id when `result: assigned` (the team this player is placed on); empty otherwise
   - `evaluator` — the staff id who recorded it (must be in `evaluators[]`), optional but preferred for accountability
   - `note` — free-text evaluator comment (human zone; the orchestrator does not judge skill)
2. The *content* of the evaluation (the soccer judgment) is the evaluator's, drawn from their role-domain (`rasa.domain.soccer.coach`; GK specialists use the live `rasa.domain.soccer.goalkeeping`). This skill records the *outcome*, never authors the assessment.
3. Validate: every `outcomes[].candidate` is in `candidates[]`; every referenced `assigned_team` resolves; every `evaluator` is in `evaluators[]`. Refuse dangling references.
4. Partial is fine — record what's known, leave the rest `pending`. Re-write, `git add`, report progress (e.g. "14 of 18 outcomes recorded; 4 pending").

### 4. Produce roster assignments + hand off to /roster

Turn outcomes into the input `/roster` consumes, then close.

1. Pre-flight: every candidate has a non-`pending` outcome. If any are still `pending`, list them and ask the user to resolve or explicitly defer before closing.
2. Group `outcomes` by `assigned_team`. Each group is a **roster assignment**: `{ team: <team-id>, players: [<member-ids where result == assigned>] }`. Waitlisted and cut candidates are recorded in the Tryout but are NOT in any assignment.
3. **Hand off to `/roster`.** Do not write team rosters from here. Surface the assignment set and invoke `/roster` (or instruct the user to) for each assigned team. `/roster` is the skill that commits players to a Roster, sets the Roster's `from_tryout` to this tryout's `id`, updates each Team's `roster_id`, and updates each member's `teams[]`. The tryout *proposes*; `/roster` *commits*. Keeping the write in one place is what preserves cross-entity integrity.
4. Once `/roster` has consumed the assignments, set the Tryout `status: complete`. Re-write the record (it now reads as the durable history of the event: who evaluated, who tried out, who landed where), `git add`, and report the close-out summary.

### Resume gracefully

Invoked against an existing record, skip the done phases and land on the next
gap. `scheduled` + 0 candidates → register. `scheduled` + candidates + partial
outcomes → record evaluations. All outcomes recorded → produce assignments +
close. Always show current state first.

## When NOT to use

- **To place players on a team without a tryout** (e.g. returning roster, manual add) → that's `/roster` directly. Not every roster comes from a tryout (`Roster.from_tryout` is optional).
- **To create or edit a member or staff record** → use `/member` / `/staff`. This skill *references* those records; it doesn't own them.
- **To create a team** → use `/team`. A tryout assigns candidates to teams that already exist.
- **To coach, train, or actually evaluate a player's ability** → that's a role-domain concern (`rasa.domain.soccer.{coach,player}`, soft; GK via the live `rasa.domain.soccer.goalkeeping`). The orchestrator records outcomes; it does not produce them.
- **To build a full Complex/Field facility record** → phase 2. This skill only stores a `location` reference and degrades gracefully when the facility entity doesn't exist yet.
- **To schedule practices or games** → phase 2 (`Schedule` entity). A tryout is a selection event, not a recurring fixture.

## What "done" looks like

- `state/tryouts/<id>.md` exists, conforms to `_template.md`, and carries a valid `age_group` + `gender` (both in `state/club.md` dimensions), a `date`, resolved `evaluators[]`, and resolved `candidates[]` — all references point at real records.
- Every candidate has a non-`pending` outcome (or pending ones were explicitly deferred by the user).
- The roster assignments were handed to `/roster`, which committed them — so each assigned member's `teams[]` and each team's `roster_id` reflect the placement (written by `/roster`, not here), and the resulting Roster's `from_tryout` equals this tryout's `id`.
- `status: complete`.
- Changes are staged (`git add`), not committed. The human reviews the diff and commits.

---

**See also:** `.claude/club-ops-rules.md` (canonical entity model + integrity
convention), `/roster` (the downstream skill this pipeline feeds), `/member`
and `/staff` (the records this skill references), `/team` (assignment targets).
Role-domains (soft): `rasa.domain.soccer.coach` (evaluator judgment),
`rasa.domain.soccer.goalkeeping` (live GK toolkit). Phase-2 entities referenced
but not authored this build: Complex / Field (location), Schedule.
