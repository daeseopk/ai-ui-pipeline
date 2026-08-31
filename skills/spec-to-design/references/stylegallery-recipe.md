# StyleGallery layout recipe

Concrete usage for Step 4's new layout-planning sub-step. StyleGallery (`changeroa/StyleGallery`) is a governed knowledge base of reusable spatial layout patterns — it answers "what structural shape does this screen need" (sidebar+content? list-detail? grid dashboard?), not "which component renders this." Keep the two separate: StyleGallery never picks a component or a color, and no design system dictates page structure.

## Which server, which tools

Two MCP servers are registered (`design-env-setup` Step 7). For layout planning, use **`stylegallery-material`** (v2, the admitted-Markdown search surface) — its 4 tools:

| Tool | Args | Use for |
|---|---|---|
| `material-discover` | — | List the 6 governed domains once, if you need to confirm `layout` exists / is `stable` |
| `material-search` | `query`, `limit`, `paths_only?` | Find candidate recipes/patterns by plain-language description |
| `material-get` | `reference` (a `stable_ref`, **not** a plain path), `offset?`, `length?` | Read the actual Markdown body of one search result |

`stylegallery` (v1, the governed claims/evidence surface — `resolve`/`claims`/`context`/`ops`/`retrieve`) is registered too but not part of this flow; it operates on named "profile" records, not the Layout domain's recipes. Don't reach for it here.

**`material-get` rejects a plain repo-relative path** (`recipes/dashboard.md` → `material_reference_invalid`). It only accepts the `stable_ref` string a `material-search` result returns (`sg:pattern/path-sha256-...` / `sg:page/path-sha256-...` / `sg:domain/...`). Always search first, take the `stable_ref` off the result you want, then `material-get` that — don't try to shortcut with a path you already know.

## The search → get flow

1. **Shortlist with `paths_only`** — cheap, fast, good for a first pass or when you're unsure of the exact term:
   ```
   material-search({ query: "list detail table filters", paths_only: true, limit: 5 })
   → paths: ["recipes/dashboard.md", "recipes/index.md", "guides/decision-tree.md", ...]
   ```
2. **Re-search without `paths_only`** once you know which title you want, to get its `stable_ref`:
   ```
   material-search({ query: "list detail table filters", limit: 5 })
   → results[].{ title, stable_ref, kind (page|pattern|domain), score }
   ```
3. **`material-get`** the chosen result's `stable_ref` to read the actual recipe body:
   ```
   material-get({ reference: "sg:pattern/path-sha256-...", length: 4000 })
   ```
   `length` caps bytes returned (max 65536) — recipes are a few KB, one call is enough. Use `offset` only if a doc is truncated and you need the rest.

If nothing in `recipes/` scores clearly for the screen, fetch `guides/decision-tree.md` the same way and work through it manually rather than guessing a recipe name.

## Turning a recipe into a screen plan

A recipe document describes semantic structure (scroll ownership, regions, composition), not markup to copy verbatim. After reading one:

1. Name the chosen recipe/pattern (e.g. "list-detail").
2. Write a 3–5 line structural outline in plain language for the specific screen — which regions exist, what each owns, what owns scrolling. Example:
   > `list-detail` — 좌측 고객 목록(자체 스크롤), 우측 상세 패널(고정 헤더 + 스크롤 바디). 리스트 상단에 검색/필터 바.
3. Map each region to real components from the list already loaded in Step 2 — the project's own design system if one exists (e.g. a `Table`/`SearchBar`/`PageFilter` for the list side, a `DetailContent` for the detail side), or Claude Design's own default components/primitives if not. This is the "map to real components" work `spec-to-design`'s Step 1 defers — it happens here, per-screen, not earlier.
4. Show the outline + component mapping to the user as the Step 4 pre-code checkpoint (see `SKILL.md`) — a short message is enough, this isn't a formal document.

## Common mistakes

- **Treating a `recipes/` hit as final without checking `primitive-to-recipe-matrix.md`** when the screen mixes two spatial problems (e.g. a dashboard with an embedded list-detail panel). Search again for the second problem rather than forcing one recipe to cover both.
- **Skipping the plan checkpoint because the recipe seems obvious.** The cost asymmetry is the whole point of this step: a wrong structural call caught here is a re-search; caught after Step 4's render checkpoint it's a rewritten `.dc.html`.
- **Letting StyleGallery guidance leak into visual decisions.** If a recipe's example markup mentions color, shadow, or animation, that's illustrative noise, not a constraint — the project's design system (its own conventions doc, if one exists) owns that, or plain semantic CSS if there's no design system.
