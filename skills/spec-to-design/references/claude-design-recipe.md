# Claude Design build recipe

Concrete patterns for Step 4 of `spec-to-design`, distilled from real end-to-end runs against a project's own design system (first pass 2026-08-11, corrected against the live product 2026-08-25 — the format below is what `get_claude_design_prompt` actually returns today; the React+Babel-CDN boilerplate this doc used to show was already stale by the second run). Everything here applies whether or not the project has a design system — where a design-system-specific bundle is involved, that's called out explicitly; the base `.dc.html`/`support.js` mechanics are the same either way.

## Page boilerplate — the real Design Components (`.dc.html`) format

**Don't reuse a React+Babel-CDN boilerplate from an earlier session without re-checking.** Claude Design's actual current file format is the `<x-dc>` template + `support.js` runtime + `DCLogic` class, not client-side Babel transpiling inline JSX. Always get the current shape from `get_claude_design_prompt`'s own output (it includes the full spec) rather than copying the skeleton below verbatim long-term — but as of this writing:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <script src="./support.js"></script>
  </head>
  <body>
    <x-dc>
      <helmet data-dc-atomics>
        <!-- The three tags below only apply if a design system is in play (spec-to-design Step 2/3).
             With no design system, skip them and build with plain markup / Claude Design's own primitives. -->
        <link rel="stylesheet" href="_ds/<folder>/_ds_bundle.css">
        <link rel="stylesheet" href="_ds/<folder>/styles.css">
        <script src="_ds/<folder>/_ds_bundle.js"></script>
        <style>/* page-local CSS */</style>
      </helmet>

      <div id="ds-root" style="min-height:100vh"></div>
    </x-dc>
    <script type="text/x-dc" data-dc-script data-props='{"$preview": {"width": 1920, "height": 1080}}'>
      class Component extends DCLogic {
        componentDidMount() {
          var Button = window.<BundleGlobal>.Button; // exact global per get_claude_design_prompt, design-system runs only
          ReactDOM.createRoot(document.getElementById('ds-root')).render(
            React.createElement(Button, { variant: 'primary' }, '검색')
          );
        }
        renderVals() { return {}; }
      }
    </script>
  </body>
</html>
```

Key differences from the old boilerplate: no React/ReactDOM/Babel `<script>` tags to add — `React`/`ReactDOM`/`DCLogic` are already injected by `support.js`. `create_support_js` must be called once per directory before any `.dc.html` in it (Step 3). The `<script type="text/x-dc" data-dc-script>` body is **plain classic JS — no `import`/`export`, no JSX**; build elements with `React.createElement(Component, props, ...children)`.

**`_ds/<folder>/` bundle path** (design-system runs only): `list_files(project_id, "_ds")` on a fresh project returns empty — the bound copy doesn't auto-materialize. `copy_files` the three assets in from the design-system project yourself once per project (Step 3 already does this; the destination convention is `_ds/<design-system-name>/{_ds_bundle.js,_ds_bundle.css,styles.css}`), then reference that path from every screen's `<helmet>`.

## Mounting a real design-system component — use `ReactDOM.createRoot`, not `x-import`

The DC template's `{{ }}` holes and `<sc-for>`/`<sc-if>` are for markup *you* compose from primitives — they are not a general way to mount an entire pre-built third-party component tree (e.g. a `Sidebar`, a `Table`) coming off a design-system bundle global. Two things that look plausible both fail:

- `<x-import component="X" from="./X.jsx">` pointing at a sibling file that does `import .../export default X` — the runtime evaluates `.jsx` files as **plain non-module scripts**. Any `import` throws `Cannot use import statement outside a module`; any `export` throws `Unexpected token 'export'`. There is no working module-export contract found for this path yet — don't reach for it for a whole-component mount.
- Returning `React.createElement(...)` from `renderVals()` as a hole value — the doc's own guidance is explicit that this is for a state-preserving leaf element, "never for layout," since the editor can't click into it.

**What actually works** — exactly the pattern `get_claude_design_prompt`'s own design-system section shows: mount into a **dedicated plain `<div id="...">`** placed directly in the template (not inside a `{{ }}` hole), and in `componentDidMount()`:

```js
componentDidMount() {
  var Sidebar = window.<BundleGlobal>.Sidebar;
  ReactDOM.createRoot(document.getElementById('ds-root')).render(
    React.createElement(Sidebar, { menuSections: menuSections, variant: 'light' })
  );
}
```

This runs as a second, independent React tree alongside the DC template's own — that's intentional (the design-system-context prompt's own words: "Mount into a dedicated child node ... not the host page's own React root, so the two trees don't collide"). The mounted subtree isn't click-editable in the DC editor; that's an acceptable tradeoff for a real design-system component you wouldn't hand-edit anyway.

## A whole class of components can't live-render in Claude Design at all

Any component that unconditionally calls a client-side router's hook (React Router's `useNavigate`/`useLocation`, or the equivalent in whatever router the project's design system depends on) throws a "must be used inside a Router" error no matter how it's mounted. Root cause: the design-system bundle's own internal router copy and any Router you could wrap around it from the page are different module/Context identities, so wrapping in a memory-router provider does not fix it. This is a converter-level gap (the router package isn't in the shared bundler's externals set), not something fixable per-screen — and it isn't specific to any one design system; it hits any component built this way.

**When you hit this**: don't spend the budget fighting it. Fall back to a hand-authored static mockup — plain markup that visually matches the real component's DOM shape and uses the same props/data you'd pass the real component, with a one-line note in the page (`정적 mockup — Claude Design 라우터 Context 제약`) so nobody mistakes it for the live component. The real render is what the eventual source/app step produces (`design-to-code`), where a genuine Router already wraps everything.

## Canvas composition — multi-screen pages

For a spec with more than one screen, use `<meta name="design_doc_mode" content="canvas">` (Step 4) and lay screens out with these rules:

1. **1920×1080 per frame.** Draw every screen at this size regardless of the eventual product's real breakpoint — it's the shared reference resolution for this pipeline's screens, keeps frames comparable side by side, and keeps a multi-screen canvas readable when the user (or a teammate) reviews it directly in the Claude Design editor. State explicitly if a screen is deliberately not full-viewport (a modal, a card) rather than silently using a different size.
2. **One frame per screen.** Don't combine multiple screens' content into a single frame's DOM — each screen is its own absolutely-positioned frame block directly inside `<x-dc>` (per the base format's canvas rules), with its own `data-screen-label`.
3. **Place flow-connected frames on the same row.** If a new screen is reached by a user action on an already-placed screen (a click, a submit, a nav item), position it in the *same row* as that screen, to its right — the canvas then reads left-to-right as the user's path, and a new row starts only for a screen that begins an unrelated flow. Concretely, with 1920×1080 frames: same-row siblings step `left += 2000` (1920 + 80px gap) at the same `top`; a new unrelated flow resets `left: 0` and steps `top += 1200` (1080 + label height + 100px gap). Compute the outer `$preview.width`/`height` as the bounding box of every placed frame once the layout is known, not a fixed guess.

## Prop mapping — verify, don't assume

The design-system guide's example composition is a starting point, not a full prop reference. Before using a component you haven't used yet this session, `read_file` its `.d.ts`:

```
read_file(project_id: <design_system_id>, path: "components/<category>/<Name>/<Name>.d.ts")
```

Real drift examples seen in practice: a story/example using `count` when the shipped prop is `badgeCount`; `currentPage` when the shipped prop is `page`. The `.d.ts` is generated from the actual shipped types — trust it over adjacent docs, stories, or memory from a previous session (the library version may have moved).

Component **variant/size prop values** (e.g. `Button`'s `variant: "primary" | "secondary" | "danger" | ...`) are an enum — pass one of the literal values from the `.d.ts`, not a guessed string.

## Styling — semantic tokens only (design-system runs)

The bundled CSS only contains rules for classes the library's own source actually uses. Arbitrary Tailwind utility classes (`bg-blue-600`, etc.) that aren't already used internally render invisible — there's no compiled rule for them. Use the design system's own semantic vocabulary (its own named colors/tokens, component `variant`/`size` props) instead of reaching for raw utility classes. If you must add layout-only CSS (flex containers, spacing between elements that aren't part of a component), plain inline `<style>` in the page head is fine — that's not competing with the bundle's compiled classes.

**The same tree-shaking hits raw CSS custom properties, not just utility classes.** A `var(--some-token)` referenced in an inline `style` or your own `<style>` block looks safe — it's not a Tailwind class, so the "arbitrary utility class renders invisible" warning above feels like it shouldn't apply — but the compiled bundle CSS only ships the custom properties that some component in the library's own source actually uses internally; the rest of the source token palette can be dropped entirely, and referencing a dropped one resolves to nothing (the property is simply unset, so anything relying on it renders blank/invisible with zero console warning). Before using any custom-property token, grep the actual compiled CSS for it (`grep -o -- "--<token-name>:[^;]*;" <bundle>/_ds_bundle.css`, or read the copied `_ds/<folder>/_ds_bundle.css` in the Claude Design project) — don't trust the source token file alone. If a design system's own conventions doc exists (see `design-to-code/references/component-mapping.md`), check there first — it may already list which tokens actually made it into the compiled bundle.

## Verifying a render

1. `render_preview(project_id, path)` → `serve_url` + `open_url`.
2. Open `serve_url` in a browser tool and wait briefly before screenshotting — the current format has no client-side Babel step, so this is faster than the old boilerplate, but still give a mounted-component render a couple of seconds before concluding something's wrong.
3. Screenshot. A blank/white page after a short wait usually means the mount target `<div id="...">` wasn't found or `componentDidMount` threw — check the console before assuming it's a timing issue (unlike the old Babel-CDN format, there's no multi-script-load race to blame by default).
4. Read console messages. There is no longer a harmless expected warning to filter out — any console message on a clean run is worth looking at. A "must be used inside a Router" error means you hit the router-hook class of components above, not a bug in your markup.
5. Fix locally, `write_files` again (unconditional overwrite is fine mid-iteration; this isn't a shared file yet), re-render, re-check.

## Common breakage and fast fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Blank white page, no console error | Mount `<div id>` id mismatch between template and `componentDidMount`, or the div rendered with zero height | Check the id strings match exactly; give the mount div `min-height` |
| `Cannot use import statement outside a module` / `Unexpected token 'export'` | Tried `import`/`export` in a sibling `.jsx` via `x-import` | Switch to `ReactDOM.createRoot` + `componentDidMount` (see above) — don't debug the import syntax further |
| A "must be used inside a Router" error | The component unconditionally uses a client-side router hook | Known converter-level gap — fall back to a static mockup (see above), don't try to wrap a memory-router provider |
| Button/label text wraps oddly at a small size | A `size="sm"` (or similar) variant is too narrow for the label | Use the default size, or add `style={{ whiteSpace: "nowrap" }}` (an explicitly supported prop) rather than fighting it with a guessed utility class |
| A color/utility class visibly does nothing | Class isn't compiled into the bundle's CSS | Swap to a semantic token or component variant from the design-system guide |
| Table/column data doesn't render as expected | Column config shape guessed instead of read from source | Check the component's real `.d.ts`/example for the exact column/row shape expected |
| `TypeError: Cannot read properties of undefined (reading '<Component>')` at the very first `window.<Global>.X` access, with the two CSS `<link>`s loading fine | The `<script src="_ds/<folder>/_ds_bundle.js">` tag itself is missing from `<helmet>` — easy to type the two `<link>` CSS tags and forget the JS one, since the page still paints (unstyled) without it | Check the helmet has all three tags, not just the two `<link>`s; confirm via a network-requests read that `_ds_bundle.js` was actually requested, not just the CSS files |
| A `Modal`/overlay component floats over neighboring frames (or the whole viewport) instead of staying inside its own frame in canvas mode | The component uses `position:fixed`, which centers on the browser viewport, not the frame's own bounding box — canvas frames are just absolutely-positioned divs on one long page | Give the frame's mount root (e.g. the `.shell` div) any CSS `transform` (`transform:translateZ(0)` is enough) — a transformed ancestor becomes the containing block for `position:fixed` descendants per the CSS spec, confining the overlay to that one frame. Verify by scrolling to a *different* frame and confirming the modal doesn't follow. |
