# RasaOS Orchestrator · Soccer · Club

**Canonical name:** `rasa.orchestrator.soccer.club` (the **first sub-namespaced orchestrator**)
**Repo / folder:** `orchestrator-soccer-club` · **Alias:** `rasa.orchestrator.soccer-club`
**Kind:** `orchestrator` (canon Spec §6) — Pattern 3 (Hybrid / CTO-style)
**Contract:** Element Contract v1.3.0
**Version:** 0.1.0 — **foundation** (entity model + People/Org/Process skills; phase-2 concerns defined, skills later)
**Status:** Live. The soccer **club-operations platform**.

## What this is

A **Pattern 3 club-operations platform** — the soccer analog of
`orchestrator.code`. It installs into a **club's umbrella repo** and runs the
whole organization: members (players/parents), staff (coaches/directors/admins/
board), teams, rosters, try-outs, leagues, tournaments, fields, complexes,
schedules, and multi-system registrations — across age-group and gender
dimensions.

> **It manages the org; the `domain.soccer.*` role family is each person's
> playbook.** A coach managed here as *staff* uses `domain.soccer.coach`; a
> player managed as a *member* uses `domain.soccer.player` / `.goalkeeping`.
> Soft references — the platform stands alone.

## v0.1.0 ships the foundation

The full **entity model** (all 13 concerns) + the **People/Org/Process** layer:

- **Spine:** [`content/club-ops-rules.md`](content/club-ops-rules.md) — the relational data model (field sets, relationships, invariants).
- **Skills:** `/member`, `/staff`, `/team`, `/roster`, `/tryout`, `/status`.
- **Records:** `state/<entity>/<id>.md` created from per-entity templates; the club config (`state/club.md`) holds the dimensions.
- **Phase 2** (defined now, skills later): `/field`, `/complex`, `/schedule`, `/tournament`, `/league`, `/registration`.

See [`content/README.md`](content/README.md) for the full install map + the 13-concern table.

## Pattern + family

- **Pattern 3** (orchestrator-core SHAPE.md) — "a club with N teams" is the
  soccer analog of `orchestrator.code`'s "a company with N repos."
- Sits over the `domain.soccer.{player,coach,parent,director,board,team,league}`
  role family (+ the content-bearing `goalkeeping`).

## See also

- `~/rAI/rasa-os/elements/orchestrator-code/` — the engineering Pattern-3 sibling.
- `~/rAI/rasa-os/elements/REGISTRY.md` — live Element registry.
