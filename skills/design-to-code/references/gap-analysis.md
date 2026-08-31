# Component Gap Analysis

Classify every component from Step 4's mapping into exactly one of six outcomes.

```
REUSE           — an existing component already covers this, as-is
COMPOSE         — no single existing component matches, but existing ones combine to cover it
EXTEND          — an existing component almost covers it; a prop/variant addition would close the gap
CREATE_LOCAL    — nothing existing covers it, and it's only needed on this screen
CREATE_SHARED   — nothing existing covers it, and there's real evidence it'll be reused elsewhere
BLOCKED         — can't resolve yet — missing information, not missing code
```

## CREATE_LOCAL vs CREATE_SHARED

Default to `CREATE_LOCAL`. Only choose `CREATE_SHARED` when there's concrete evidence beyond this one screen — e.g. the same element already appears in another already-sourced screen, or the user explicitly says it'll be reused. **Do not promote something to a shared/common component speculatively** — that's exactly the kind of AI-invents-structure behavior this skill exists to avoid. A `CREATE_LOCAL` component can always be promoted later once a second real use case shows up; the reverse (de-promoting an over-eagerly shared component that turned out to be a one-off) is more disruptive.

## BLOCKED

Use this when the blocker is missing *information*, not missing *code* — e.g. the Design Artifact's data requirement is `Unknown` and the user hasn't answered yet, or a needed design-system component genuinely doesn't exist and the user hasn't chosen how to proceed (see `SKILL.md`'s component-gap policy: escalate to the design system's own repo, build local, or substitute with a TODO marker). Don't let a `BLOCKED` item silently become `CREATE_LOCAL` just to keep moving.

## Output shape (`templates/gap-report.md`)

```text
Missing Component

Name:
DateRangePicker

Search:
- Design system (if any): NOT FOUND
- Project Components: NOT FOUND

Recommendation:
CREATE_LOCAL

Reason:
Current Design only requires this component.
No evidence that it should become a shared component.
```

Every `CREATE_LOCAL`/`CREATE_SHARED`/`BLOCKED` entry gets one of these — a bare "doesn't exist, will create" without the search evidence and reasoning is not acceptable output.
