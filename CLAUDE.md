# CLAUDE.md — `rasa.orchestrator.soccer.club`

Per-repo working contract for Claude sessions opened inside this folder (the
Element repo itself). Extends `~/.claude/CLAUDE.md` and the workspace
`~/rAI/rasa-os/CLAUDE.md`; does not override them.

## What you are when you're in this folder

You are working on **`rasa.orchestrator.soccer.club`** — the **first
sub-namespaced orchestrator** and a **Pattern 3 (Hybrid / CTO-style)
club-operations platform** (the soccer analog of `orchestrator.code`). It
installs into a **club's umbrella repo** and runs the organization across 13
operational concerns.

## The load-bearing rule

**Two truths.** *A fact about the club's organization* (a member, a team, a
roster, a tryout) → the orchestrator owns it, in `state/`. *The work of a
role* (how a coach coaches, how a keeper trains) → a `domain.soccer.*` role
toolkit. **The orchestrator manages people, units, and events; the role
domains are their playbooks.** References to the role domains are **soft** (no
`requires.elements[]`); only `domain.soccer.goalkeeping` is content-bearing so
far.

## Source of truth

- **`~/rAI/rasa-os/elements/orchestrator-core/content/SHAPE.md`** — Pattern 3.
- **`content/club-ops-rules.md`** — the entity model + the contract every skill executes (field sets, relationships, invariants).
- **`content/README.md`** — what installs where.
- **`rasa.json`** — the declaration + install manifest.

## Don'ts

- **Don't break the integrity contract.** A record change must update every
  surface it touches or none (e.g. `/roster` updates the roster AND the team's
  `roster_id` AND each member's `teams[]`). Field names are canonical — keep
  the skills, templates, and rules in lockstep.
- **Don't author role-domain content here.** How a coach/player/etc. does
  their work belongs in `domain.soccer.<role>`, not in this orchestrator.
- **Don't harden the role-family references into `requires.elements[]`.** Soft,
  graceful-if-absent.
- **Don't `bin/init` this Element into itself.** `content/` is the source.

## How a version bump works

- **0.1.0 → 0.2.0** — the **phase-2** skills (`/field`, `/complex`,
  `/schedule`, `/tournament`, `/league`, `/registration`) + their entity
  templates. Then deterministic `bin/` helpers (script-craft).
- Patch — template/wording fixes. Major (→1.0.0) — first stable lock-down.

Each bump: `VERSION` + `rasa.json#version`, CHANGELOG, `bin/check-manifest`,
commit + tag; update `elements/REGISTRY.md` + `elements/CHANGELOG.md`.
