# Design Artifact — validation + schema

## Step 1 — Design Source Validation

Before touching any code, confirm the design source is actually usable. Which checks apply depends on `{input_mode}` from Step 0 — run the matching case below and stop on the first failure.

### Case 1 — `mcp` (live Claude Design access)

```
project_id/path parsed from the open_url the user gave you
  → get_project(project_id) succeeds
  → list_files(project_id) contains the target path
  → read_file(project_id, path) returns non-empty .dc.html content
  → any asset references inside it (images, icons) resolve
```

```
Design Source Validation (mcp)

✓ Design URL accessible
✓ Screen found
✓ Design specification found
✓ Assets accessible

Status: READY
```

### Case 2 — `local` (`.dc.html` file from `design-export`)

**File selection (before validation):** if the user hasn't already given a specific file path, search for local `.dc.html` files first — scoped to the target project directory only (excluding `node_modules`), never outside it. `design-export` writes to the current working directory by default, so a project-scoped search is where a real export normally lands; don't widen the search to the scratchpad or any other path outside the project.

- **One or more found**: show the list (filename, path, modified time) and ask the user whether to work with one of them, or add a new file to the session instead. Don't silently pick the newest one for them.
- **None found**: ask the user to add a `.dc.html` file (export a new one via `design-export`, or attach one directly) — don't proceed without one.
- Only the path the user actually confirms moves on to the check below.

```
file path selected/given by the user
  → file exists and is readable
  → content is non-empty and looks like a real .dc.html
    (has <script type="text/x-dc"> and a componentDidMount call)
```

Skip the asset check entirely — `design-export` never pulls design-system bundle assets by design (see its Step 3), so there's nothing to verify here.

```
Design Source Validation (local)

✓ File found and readable
✓ Design specification found
– Assets accessible (skipped — not applicable to a local export)

Status: READY
```

If the user gave a file with no accompanying link, leave `screen.url` as `Unknown` (see schema below) rather than asking for one just to fill the field — this only matters later if Step 10 needs a live screenshot to compare against.

### Failure (either case)

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
  id: <project_id>/<path>          # Case 1: from the open_url. Case 2: local file name/path
  name: <data-screen-label if canvas mode, else the file path>
  url: <the open_url the user gave you — never the serve_url. Case 2 with no link given: Unknown>
  version: <Case 1: read_file's returned content hash/timestamp. Case 2: file mtime, or Unknown>

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

Applies equally to Case 1 and Case 2 — this is about the `.dc.html` content itself, not how it arrived. Confirmed from actual Claude Design output (`spec-to-design/references/claude-design-recipe.md`, and this session's own P-01 build):

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
