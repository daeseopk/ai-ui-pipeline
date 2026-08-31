# Implementation Rules — Source Code Generation (Step 8)

`convention.md` is the final authority on folder structure, naming, and export policy — load it before writing anything and follow it exactly. This doc covers the rules that don't belong in `convention.md` because they're about *how this skill behaves*, not project style.

## When an existing component almost-but-not-quite fits

Work through this order before concluding a new component is needed — this is the concrete procedure behind Component Mapping's priority list:

```
1. Check the existing component's props/config — does a prop combination already produce what Design needs?
2. Check composition — do two existing components together cover it?
3. Check extension — would adding a prop/variant to the existing component close the gap cleanly,
   without breaking its other current usages?
4. Only if all three fail: new component (CREATE_LOCAL/CREATE_SHARED per gap-analysis.md)
```

## Must follow

- Existing project architecture (whatever `convention.md` + Step 5's repository re-check establish)
- Existing components first, always (see priority order above)
- Existing naming convention (`convention.md`)
- Existing styling convention (`convention.md` + `component-mapping.md`'s semantic-token rule, if a design system is in use)
- Existing state management choice per data category (`convention.md`, or CLAUDE.md §3.1's four-category mapping if `convention.md` hasn't specified one yet)
- Existing API abstraction, if one exists in the project
- Existing folder structure (`convention.md`)

## Forbidden

- Creating a new component when an existing one already does the job
- Adding functionality the Design doesn't call for
- Inventing an API endpoint or its shape
- Inventing a business rule the Design doesn't imply
- Introducing a new UI library alongside the project's existing one(s)
- Changing existing project architecture to fit this one screen
- Modifying an existing shared component just because this Design wants it slightly different — extend it (new prop/variant) instead of changing its default behavior for everyone else

## Cold start (no existing pattern yet)

If Step 5 found nothing to reuse (a genuinely first screen), don't block on "no existing pattern" — build per `convention.md`'s stated structure, and the result becomes the pattern later screens reuse. Say so explicitly in the Implementation Plan (`Unknown`/notes section): "no existing project pattern — this establishes the baseline."
