# Layout Planning

This step exists to keep one boundary intact: **StyleGallery decides spatial structure, the design system (if the project has one) decides which component fills each slot. Neither one does the other's job.**

```
Screen
 ↓
StyleGallery            → layout pattern (e.g. "sidebar + content", "list-detail")
 ↓
Component placement     → which region gets which component (Step 4, not here)
```

## Actual usage

Don't duplicate the tool-call mechanics here — `spec-to-design` already has them, and they don't change for this skill: see `../../spec-to-design/references/stylegallery-recipe.md` for the exact `material-search` → `material-get` flow, the `stylegallery-material` vs `stylegallery` server distinction, and how to turn a recipe into a structural outline.

The only difference in this skill's context: the outline you produce here feeds Step 4 (Component Mapping) and Step 6 (Gap Analysis) for a screen that **already has a committed layout** (it's sitting in Claude Design, already built) — so this step is closer to *reading back* the structural pattern the screen already embodies than picking a fresh one. Use the Design Artifact's `layout.hierarchy` (from `design-artifact.md`) as the starting point, and cross-check it against a `material-search` result rather than searching cold.

## When nothing fits

If no StyleGallery pattern matches, say so explicitly rather than forcing the nearest one:

```
Layout Pattern: no clear match in StyleGallery's recipes/
Structural outline (derived directly from the artifact instead): ...
```

Don't block Step 4 on this — an explicit "no pattern, derived directly from the artifact" is a valid outcome, a silently-wrong forced match is not.
