# Component Mapping

Map each UI element the Design Artifact calls for to a real Repository component, in this priority order:

```
1. The project's own design system (if one is registered in Claude Design and installed in the project — always check first, when it exists)
2. Existing project common component     (the project's own shared components, if any exist yet)
3. Existing feature component            (something built for a different screen, reusable as-is)
4. Composition of existing components    (no single match, but existing pieces combine to cover it)
5. New component                         (only after 1–4 are genuinely exhausted — see gap-analysis.md)
```

Never create a new component when an existing one already does the job — check props/config and composition options before concluding something is missing (see `implementation-rules.md`'s resolution order). **If the project has no design system at all, priority 1 simply doesn't apply — start at priority 2.** That's a normal, expected state, not a gap to work around.

## When a design system exists — read its own ground truth, don't re-derive it

If Step 0 found a design system, that library almost certainly has its own conventions/notes documentation from whatever sync process registered it with Claude Design (commonly a `.design-sync/conventions.md` and `.design-sync/NOTES.md`-style pair, but the exact location depends on the project). **Find and read that documentation before mapping anything** — don't trust a component's name, a `.stories.tsx` file, or memory from a previous session. It typically covers:

- The real shipped export names (which can differ from the component folder name — see below)
- The styling idiom (semantic tokens vs. raw utility classes — most design systems only compile CSS for classes their own source actually uses, so an unused utility class can render invisible with no warning)
- Known gaps: components that don't live-render inside Claude Design's sandbox, components whose public API doesn't match their apparent name, etc.

Do not hardcode any one design system's specific component names or gaps into this skill — that knowledge lives with the design system itself (or its own conventions doc) and gets re-read fresh each run, since it can drift between runs.

## A component folder existing does not guarantee that name is what gets exported

Real drift patterns seen across design systems in practice: a component's folder/display name doesn't match its actual export (e.g. a "Toast" folder that only exports a `ToastProvider`; a single "Uploader" concept implemented as three separate exports for different upload modes; a component whose imperative behavior is exported as a function, with no corresponding UI component at all — a genuine public-API gap, not a documentation gap). Always confirm the real export name against the design system's actual index/entry point or its own conventions doc, not the folder name.

## Known Claude-Design-sandbox limitation that does NOT apply once sourcified

Any component that unconditionally calls a client-side router's hook can't live-render inside Claude Design's sandbox (see `spec-to-design/references/claude-design-recipe.md`'s "A whole class of components can't live-render in Claude Design at all"), so `spec-to-design` had to substitute a static mockup for it.

**That constraint does not exist in the real target-project app** — assuming the target app has (or will have) a genuine Router wrapping the whole tree. If a screen's artifact contains a static-mockup substitute for such a component (look for a comment like `정적 mockup — Claude Design 라우터 Context 제약`), **do not copy that markup into source.** Use the real component instead — this is the one case where the artifact's own markup is known to be intentionally wrong and should be overridden, not trusted literally.

## Verify props before using any component you haven't used yet this run

Even after confirming the right export name, `read_file` its `.d.ts` (from the design-system's Claude Design project) or the shipped type declarations once installed locally, before writing props — story/example files have been known to drift from shipped APIs (a story using one prop name when the real shipped prop is different).

## Example mapping table (illustrative — actual entries always come from Step 5's real repository check, never assumed)

| Design | Repository | Result |
|---|---|---|
| Button | design system's `Button` | Reuse |
| Select | design system's `Select` | Reuse |
| Table | design system's `Table` | Reuse |
| UserStatusBadge | none found | New (see gap-analysis.md) |
| SearchArea | no single match, but existing `SearchBar` + `PageFilter` combine to cover it | Compose |

## No design system

If Step 0 confirms there's no design system at all, every row in this table starts from priority 2 (existing project components) — build with the project's own established primitives (or, on a true cold start, plain semantic HTML + the project's styling convention from `convention.md`). This is a valid, complete path through this step, not a degraded one.
