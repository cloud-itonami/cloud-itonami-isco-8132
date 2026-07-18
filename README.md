# cloud-itonami-isco-8132

Open Occupation Blueprint for **ISCO-08 8132**: Photographic Products Machine Operators.

This repository designs a forkable OSS business for a photographic products (film/photo) processing plant scheduling and logistics coordination practice: a plant scheduling and supply-coordination robot manages crew/task records under a governor-gated actor, so a film/photo processing plant crew keeps its own operating records instead of renting a closed workforce-management SaaS.

**Maturity: `:implemented`.** `src/photolab/` implements the
`PhotoLabCoordActor` as a `langgraph.graph/state-graph`
(`photolab.actor`) wired to a `Photographic Products Machine
Operations Coordination Advisor` (`photolab.advisor`) and an
independent `PhotoLabCoordGovernor` (`photolab.governor`), following
the itonami actor pattern (ADR-2607121000): `:intake -> :advise ->
:govern -> :decide -+-> :commit (:ok?) +-> :request-approval
(:escalate?, human-in-the-loop interrupt) +-> :hold (:hard?)`. 23
tests / 49 assertions green (`clojure -M:test`). HARD
invariants (always hold, never overridable): operator provenance,
plant provenance, no-actuation (`:effect` must be `:propose`), a
closed op-allowlist (`:log-work-record`, `:schedule-crew-operation`,
`:flag-safety-concern`, `:coordinate-supply-order` — nothing else may
ever be proposed), and a permanent, unconditional block on any
proposal that would directly finalize a processing-operation-execution
decision or a chemical-safety-clearance decision, or override a plant
safety officer's judgment. Always-escalate paths (human sign-off
regardless of confidence, mapping this repo's Trust Controls in
[`docs/business-model.md`](docs/business-model.md)):
`:flag-safety-concern` (always) and `:coordinate-supply-order` above
the registered cost threshold.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a plant scheduling/logistics coordination robot performs crew scheduling, production-run/inventory/progress-record logging and darkroom-chemical/film-stock supply-order coordination for a photographic products (film/photo) processing plant crew, under an actor that proposes actions and an independent **Photographic Products Machine Operations Coordination Governor** that gates them. The governor never
dispatches hardware itself, never operates film/photo-processing equipment (developer/fixer processing lines, rollers, processing machinery) on the plant floor, and never finalizes a processing-operation-execution decision or a chemical-safety-clearance decision, nor overrides a plant safety officer's judgment; `:high`/`:safety-critical` actions (such as a flagged chemical-exposure/equipment-condition concern, or an above-threshold supply order) require human sign-off. **This actor coordinates plant scheduling/logistics only — it never operates film/photo-processing equipment or makes chemical-safety-clearance decisions itself.**

## Hazard context

Photographic Products Machine Operators run film/photo-processing equipment using darkroom chemicals (developer, fixer, and historically some toxic silver-based/chromium compounds) — a chemical-exposure hazard, plus an equipment hazard from rollers and processing machinery. This actor never directly finalizes a processing-operation-execution decision or a chemical-safety-clearance decision, and never overrides a plant safety officer's judgment: those decisions stay the plant safety officer's exclusive, permanent authority.

## Core Contract

```text
crew roster + plant registration + safety-reporting policy
        |
        v
Photographic Products Machine Operations Coordination Advisor -> PhotoLabCoordGovernor -> log/schedule/coordinate, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, finalize
a processing-operation-execution decision, declare a chemical-safety-clearance decision,
override a plant safety officer's judgment, suppress an operating record, or
disclose sensitive data without governor approval and audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `8132`). Required capabilities:

- :robotics
- :identity
- :audit-ledger

See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md).

## License

AGPL-3.0-or-later.
