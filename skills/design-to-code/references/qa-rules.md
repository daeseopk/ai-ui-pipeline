# QA Rules — Step 9/10

## Step 9 — Build / Typecheck / Lint (must pass before QA)

```bash
pnpm typecheck && pnpm lint && pnpm build
```

Any failure stops here — fix and re-run before moving to Step 10. Don't screenshot a build that doesn't compile.

## Step 10 — Visual / Functional QA

Run `pnpm dev`, open the real route in a browser tool, and compare against the Claude Design `open_url` screenshot for the same screen. Check:

```
Layout
Spacing
Typography
Component hierarchy
Size
Alignment
Responsive behavior
States (whichever ones Step 2 confirmed — don't invent ones that weren't specified)
```

Also check the browser console for real errors — a real target-project app has a genuine Router and real design-system imports (if one is in use), so none of `spec-to-design`'s known Claude-Design-sandbox warnings (the router-hook Context gap, etc.) should appear here. Any console error at this stage is a real bug, not an expected artifact of the environment.

## Result format

```text
Visual QA

✓ Layout structure
✓ Component hierarchy
✓ Responsive layout

⚠ Search input width differs
⚠ Table row height differs

✗ Mobile empty state missing
```

`✓` = matches. `⚠` = close but a real, specific difference — always name it, never a vague "looks slightly off." `✗` = missing entirely or functionally broken. A screen with any `✗` is not done — go back to Step 8 for that item before writing the Implementation Report.

For a multi-screen run, repeat this checkpoint per screen and get user confirmation before starting the next one (same pattern `spec-to-design` uses).
