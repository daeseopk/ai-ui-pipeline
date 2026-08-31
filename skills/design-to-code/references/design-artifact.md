# Design Artifact — validation + schema

## Step 1 — Design Source Validation

Before touching any code, confirm the Claude Design screen is actually usable as a source. Run these checks in order and stop on the first failure:

```
project_id/path parsed from the open_url the user gave you
  → get_project(project_id) succeeds
  → list_files(project_id) contains the target path
  → read_file(project_id, path) returns non-empty .dc.html content
  → any asset references inside it (images, icons) resolve
```

Report the result in this exact shape:

```
Design Source Validation

✓ Design URL accessible
✓ Screen found
✓ Design specification found
✓ Assets accessible

Status: READY
```

or

```
✗ Design specification unavailable

Status: BLOCKED

Reason:
Implementation cannot safely proceed without design specification.
```

Do not proceed to Step 2 on a `BLOCKED` result. Don't retry the same call hoping it resolves itself — report the blocker to the user.

## Design Artifact schema

A Claude Design screen is not just a URL — treat it as a structured artifact with these fields. Fill every field from what `read_file` actually returns; **never invent a value the artifact doesn't provide.** Use the literal string `Unknown` or `Needs Clarification` where information genuinely isn't there — that's the expected, correct value in those cases, not a failure.

```yaml
screen:
  id: <project_id>/<path>          # from the open_url
  name: <data-screen-label if canvas mode, else the file path>
  url: <the open_url the user gave you — never the serve_url>
  version: <read_file's returned content hash/timestamp, if available>

  layout:
    hierarchy: <React.createElement tree under componentDidMount — component + prop + child nesting>
    canvas_position: { left, top }  # only present in canvas mode; used for flow inference in Step 3/5 of layout planning, not here

  components:                       # [{ name, props }] — literal values as written in the .dc.html
  data:
    static_copy: [...]              # literal, non-repeating text — treat as real copy
    mock_candidates: [...]          # repeated similar objects (e.g. table rows) that look like sample data

  behaviors: Needs Clarification    # a static .dc.html cannot show interaction timing/logic — always draft this with the user, never assume
  states: Needs Clarification       # a Claude Design frame is one static state; loading/empty/error/etc. are not derivable from it alone
```

## What is and isn't extractable from a real `.dc.html`

Confirmed from actual Claude Design output (`spec-to-design/references/claude-design-recipe.md`, and this session's own P-01 build):

**Extractable directly:**
- Which components are mounted (`window.<BundleGlobal>.X` references, if a design system is in use) and the literal props passed to each
- The composition/nesting of those components (the `React.createElement` tree)
- The screen's name/label and, in canvas mode, its position relative to other screens
- Literal static text

**Not extractable — always `Unknown`/`Needs Clarification` until the user fills it in:**
- Real API endpoints or data-fetching specifics
- Interaction behavior (debounce timing, validation rules, what a click actually does beyond navigation)
- Non-default UI states (loading/empty/error/disabled/hover) — the artifact shows exactly one static state
- Whether a repeated-looking data block is real static copy or a placeholder for server data — flag it as a `mock_candidate` and ask, don't guess

If `spec-to-design`'s output later gets an explicit mock-data marking convention, prefer that marker over guessing from repetition — but until then, treat visually-repeated near-identical objects as `mock_candidates` requiring confirmation.
