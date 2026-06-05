# Orchestrator — Spec-Driven Design Supervisor

A plugin that adds a **Supervisor skill** and a set of **self-contained phase skills** which orchestrate
the eight specialist skills in the sibling [`plan`](../plan) plugin into a single, gated,
*spec → plan → build → ship* lifecycle.

The Supervisor does no work itself. It **routes** between phases based on files on disk, **enforces**
an adversarial validation gate at every handoff, and is the **sole git authority**. Delegation is
**runtime-agnostic** — the protocol hands off *file paths*, so it runs unchanged whether skills are
activated by Gemini (`activate_skill`) or Claude Code (the `Skill` / `Agent` tools).

## Why it exists

The `plan` plugin grew from 4 roles to **8 skills** — adding `spec_validator`,
`adversarial-plan-validation`, `adversarial-implementation-validation`, and `simplifier`. Those
validators are designed to sit *between* phases ("after a spec, before a plan"; "after a plan, before
executing"; "after code, before merge"), but nothing dispatched them. This plugin wires them into the
seams.

## The enriched lifecycle

```mermaid
graph TD
    Start(["User Request"]) --> R["Phase 0 research → context.md"]
    R --> PO["Phase 1 discovery → product_owner → spec.md"]
    PO --> SV{"🔎 spec_validator gate"}
    SV -- confirmed findings --> PO
    SV -- clean --> ARCH["Phase 2 planning → architect → plan.md"]
    ARCH --> PV{"🔎 plan-validation gate"}
    PV -- confirmed findings --> ARCH
    PV -- clean --> Human{"🛑 Human approval (spec + plan)"}
    Human -- reject --> ARCH
    Human -- approve --> ENG["Phase 3 construction → engineer → simplifier"]
    ENG --> AUD{"auditor"}
    AUD -- code broken --> ENG
    AUD -- plan wrong --> ARCH
    AUD -- pass --> IV{"🔎 implementation-validation gate"}
    IV -- confirmed defects --> ENG
    IV -- clean --> Git{"🛑 Git gate (commit per group)"}
    Git -- approve --> CheckRelease{"Release complete?"}
    CheckRelease -- no --> ENG
    CheckRelease -- yes --> Rel["Phase 4 release → 🛑 tag & ship"]
```

## The skill set

| Skill | Phase | Role | Delegates to (`plan` plugin) |
|---|---|---|---|
| `supervisor` | — | Router, gatekeeper, git authority. **Start here.** | (all, via the phase skills) |
| `research` | 0 | Investigate the codebase → Context Report | *(built-in investigation)* |
| `discovery` | 1 | Author + validate the spec | `product_owner` → **`spec_validator`** |
| `planning` | 2 | Author + validate the plan | `architect` → **`adversarial-plan-validation`** |
| `construction` | 3 | Implement, refine, audit, attack, commit | `engineer` → `simplifier` → `auditor` → **`adversarial-implementation-validation`** |
| `release` | 4 | Tag & ship | `product_owner` |

All eight `plan` skills are exercised, none orphaned. The four cross-cutting **contracts** (artifact map,
delegation contract, gate semantics, git authority) are defined once in `supervisor/SKILL.md` and
referenced by the phase skills.

## Artifact map (single source of truth)

```
plans/00-ROADMAP.md                                  # master roadmap (product_owner)
plans/research/{topic}_context.md                    # research report
plans/active_milestones/{moniker}/context.md         # moved-in research
plans/active_milestones/{moniker}/spec.md            # product_owner
plans/active_milestones/{moniker}/plan.md            # architect (+ data-model.md / api-contracts.md)
plans/active_milestones/{moniker}/validation/        # gate verdicts (this plugin's convention)
    spec-validation.md  plan-validation.md  impl-validation.md
plans/audit/AUDIT_{plan}.md                          # auditor (keep plans/audit/.gitignore = *)
```

Because every phase writes to fixed paths and declares a precondition → exit-gate, the Supervisor
re-derives the current phase from disk each turn. The pipeline is therefore **resumable** — interrupt it
and ask the Supervisor to "continue", and it picks up at the earliest unsatisfied gate.

## Validation gates (default-on, skip-for-trivial)

Three adversarial gates are **mandatory for complex milestones**, in order: **spec → plan →
implementation**. Each runs a validator skill (an independent 3-skeptic panel with a 2-of-3 majority),
persists its verdict to `validation/*.md`, and loops back to the producer on any *confirmed* finding
(re-running once). *Unconfirmed* findings are surfaced, never silently dropped.

**Trivial fast-path:** a typo / one-line / no-edge-case change may bypass grilling and all three gates,
going straight to a minimal plan → `engineer` → `auditor`. When in doubt, treat the milestone as complex.

Two **human checkpoints** plus the git gate are never skipped: 🛑 approve spec+plan before building,
🛑 approve every per-group commit, 🛑 approve the release tag.

## Install & activate

This is a sibling plugin to `plan`; it **requires the `plan` plugin's skills to be present** (it
delegates to them by name). Skills auto-discover from the `skills/` directory — the manifest is just:

```json
{ "name": "orchestrator" }
```

To drive a piece of work end to end, **invoke the `supervisor` skill** (or ask the assistant to "act as
the orchestrator Supervisor and drive this from idea to commit"). The Supervisor detects state and routes
to the correct phase, stopping at each human/git checkpoint.

> For the Gemini Swarm runtime, the legacy `system.md` Supervisor (installed via `/swarm:init`) remains
> available; this plugin is the skill-native, validator-enriched evolution of that protocol.
