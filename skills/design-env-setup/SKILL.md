---
name: design-env-setup
description: Set up and verify the local environment for the Claude Code → Claude Design pipeline — the claude-design MCP server, the Claude Design product-access consent, and the StyleGallery layout-planning MCP. Optionally syncs a project's own component library into Claude Design as a design system, if one exists. Use when the user asks to set up, verify, or fix this environment, or when spec-to-design reports a broken prerequisite.
metadata:
  author: 김대섭
  version: "2.0.0"
  status: "local-only draft — not yet packaged for org-wide distribution"
---

# design-env-setup

Brings a fresh Claude Code session to the point where the `spec-to-design` skill can run: the `claude-design` MCP server is registered, Claude Design product access is approved, and — if the project has its own component library it wants Claude Design screens built from — that library's design system is synced and published. **A design system is optional.** Without one, `spec-to-design` still works, just building screens from Claude Design's own default components instead of a project-specific library.

Run every step in order. Steps 1–2 only apply if the project has a design system it wants synced; skip them (and their checklist items) otherwise. Each remaining step has a command-based check where one exists; where Claude Code cannot check something itself (a web UI toggle), stop and ask the user to confirm before moving on — do not assume.

## Prerequisites

- A Chrome browser tool (`mcp__claude-in-chrome__*`) available for the steps that require the Claude Design web UI.
- If the project has a design system to sync: that component library's repo cloned locally. Ask the user for the path if unknown — do not guess a path, and do not assume one exists.

## Step 0 — Auth check

```bash
claude auth status
```

Expect `"loggedIn": true, "authMethod": "claude.ai"`. If not logged in, tell the user to run `claude auth login` themselves — do not attempt to log in on their behalf.

## Step 1 — `/design-sync` (component library → Claude Design) — optional

**Skip this step entirely if the project has no component library it wants synced into Claude Design.** `spec-to-design` works without a design system — it just builds with Claude Design's own default components instead.

If there is one: `/design-sync` operates on **the repo of the directory it's run in** — this is the most common mistake. Confirm the working directory is that component library's repo root before running it:

```bash
cd <component library repo path>
```

Then run `/design-sync` in that session. On first run it shows a design-system access consent prompt — this grants the `designOauth` credential (scopes `user:design:read`/`user:design:write`), separate from everything in Step 3. Approve it.

If the session has no claude.ai login (rare — pure API-key CLI sessions), run `/design-login` first; it grants that same `designOauth` credential through a different route.

Check the result:

```bash
ls <component library path>/.design-sync/
```

Expect `config.json`, `NOTES.md`, `conventions.md`, `previews/`. If `.design-sync/` already exists and is committed, this is a re-sync — read `NOTES.md` first so you inherit prior fixes (component name-mapping, CSS gotchas) instead of rediscovering them.

## Step 2 — Publish the design system (web UI only, no API path) — optional, only if Step 1 ran

Claude Design has no API/MCP method to publish a design system or set an org default — confirmed by testing, not just docs. This step is manual:

1. Open `https://claude.ai/design`.
2. Click the org name (bottom-left of the project list).
3. Upload brand assets if not already done → Claude extracts the design system.
4. Toggle **Published** on for the design system that Step 1 produced.

Verify by navigating to the **Design systems** tab and checking that design system's row shows the `Published` check and (if it should be the account/org default) the `Org default` badge. If the window is narrow, the `Published`/`Access` columns collapse via responsive CSS and won't screenshot — widen the window (`resize_window`) rather than forcing the CSS, or the screenshot will show a state the user can't actually see either.

Ask the user to confirm this step is done before proceeding — there is no command-line signal for it.

## Step 3 — Register the `claude-design` MCP server

```bash
claude mcp add --scope user --transport http claude-design https://api.anthropic.com/v1/design/mcp
claude mcp list
```

Expect `claude-design: https://api.anthropic.com/v1/design/mcp (HTTP) - ✔ Connected`.

**This "Connected" status is a transport-level health check only — it does not mean the server's tools are usable.** Do not stop here. `claude mcp login claude-design` does nothing useful for this server (it replies that auth is automatic via the Claude login) — don't waste a step on it.

## Step 4 — Approve Claude Design product access (the step people actually miss)

Registering the MCP server is not enough. Calling any of its tools without this returns `{"error":"needs_consent","consent":"agent_design_projects", ...}` — a completely separate grant from Step 1's `designOauth`, tied to the account's main OAuth (`claudeAiOauth`, scope `user:mcp_servers`).

1. Open `https://claude.ai/design/settings`.
2. Set **Claude product access** to **On**.
3. New tool calls that write to a project add an entry under **Project write access** below it, with a per-project **Revoke** button — that's the standing write grant, not something to set up separately.

Ask the user to confirm this is done — there is no CLI signal for it either.

## Step 5 — Verify the MCP tools are actually callable

Restart the Claude Code session (this step, like the MCP server itself, is session-scoped — a server added or a consent granted mid-session does not retroactively appear in that session's tool list).

After restart, check whether the tools loaded:

```
ToolSearch query="select:mcp__claude-design__list_design_systems"
```

- **If found**: call it. A real response (a JSON array, possibly empty if no design system was synced) means the whole pipeline is live — `spec-to-design` can run normally.
- **If not found** (`No matching deferred tools found`), even after a restart and Steps 3–4 both being genuinely done: this is a known client-side bug where a `claude mcp add`-registered server's tools don't get wired into some sessions even when the server itself is healthy and authorized. Do not keep restarting hoping it resolves — fall back to the raw-MCP workaround below, which talks to the exact same server and account, just bypassing Claude Code's own tool-loading path.

### Fallback: raw MCP HTTP calls (only if Step 5's ToolSearch comes up empty)

The server is a normal streamable-HTTP MCP endpoint; Claude Code's own stored OAuth token can call it directly:

```bash
TOKEN=$(security find-generic-password -a "$USER" -s "Claude Code-credentials" -w \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['claudeAiOauth']['accessToken'])")

curl -s -X POST "https://api.anthropic.com/v1/design/mcp" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"list_design_systems","arguments":{}}}'
```

Never print `$TOKEN` itself into any response or file — pipe it straight into the request and `unset` it after. If this also returns `needs_consent`, Step 4 was not actually completed — go back, don't retry the curl.

If this fallback is what's actually working, `spec-to-design` must use it too instead of assuming the native `mcp__claude-design__*` tools exist — check which mode you're in before starting that skill, and say so to the user rather than silently switching approaches mid-task. This one-off `list_design_systems` curl is only for the verification check in this step; for the full set of calls `spec-to-design` actually needs (`create_project`, `write_files`, `render_preview`, etc.), use the reusable helper and per-tool argument guidance in `spec-to-design/references/mcp-fallback-recipe.md` rather than hand-rolling a new curl per tool.

## Step 6 — Register the StyleGallery MCP servers

`spec-to-design`'s Step 4 layout-planning sub-step needs the [StyleGallery](https://github.com/changeroa/StyleGallery) MCP tools to classify a screen and pick a layout recipe before writing any `.dc.html`. Register both packaged servers — they're local stdio processes launched via `npx`, no account/OAuth involved:

```bash
claude mcp add --scope user stylegallery -- npx --yes --package stylegallery stylegallery-mcp
claude mcp add --scope user stylegallery-material -- npx --yes --package stylegallery stylegallery-material-mcp
claude mcp list
```

Expect both to show `✔ Connected`. `spec-to-design` only actually uses `stylegallery-material` (the v2 Markdown-search surface — `material-discover`/`material-search`/`material-get`); `stylegallery` (v1, governed claims/evidence) is registered for completeness but isn't part of that flow today.

Requires Node.js 22+ (StyleGallery's own requirement) — check `node -v` if the `npx` launch fails outright.

**Same session-scoped caveat as Step 3's `claude-design` server**: a server registered mid-session doesn't retroactively appear in that session's tool list. After registering, restart the session and verify:

```
ToolSearch query="select:mcp__stylegallery-material__material-search"
```

Unlike `claude-design`, this server has no consent/OAuth gate sitting behind the transport-level "Connected" status — if `claude mcp list` shows Connected and a post-restart `ToolSearch` still comes up empty, that's a genuinely new failure mode worth investigating fresh, not the same known bug documented in Step 5.

## Final checklist

- [ ] `claude auth status` → `loggedIn: true`
- [ ] *(only if a design system is being synced)* `<component library path>/.design-sync/` exists
- [ ] *(only if a design system is being synced)* `claude.ai/design` → Design systems → the project's design system shows `Published`
- [ ] `claude mcp list` → `claude-design` Connected
- [ ] `claude.ai/design/settings` → Claude product access `On`
- [ ] Post-restart: `mcp__claude-design__list_design_systems` either callable directly, or confirmed working via the raw-HTTP fallback — state which mode is active
- [ ] `claude mcp list` → `stylegallery`, `stylegallery-material` both Connected
- [ ] Post-restart: `mcp__stylegallery-material__material-search` actually loads via `ToolSearch`
