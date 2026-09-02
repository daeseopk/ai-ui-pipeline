---
name: design-export
description: Pull a Claude Design screen's raw .dc.html source (and, on request, the whole project's screens) out to a local file, so it can be handed to someone who doesn't have Claude Design project access — most commonly another developer's Claude Code session that will feed it into design-to-code. Use when the user has a Claude Design screen/project they made and wants to export its source for sharing, rather than inviting the recipient as a project member.
metadata:
  author: 김대섭
  version: "1.0.0"
  status: "local-only draft — not yet validated end-to-end"
---

# design-export

Read-only, one-way extraction: take a Claude Design screen the *current* account already has access to, and write its actual `.dc.html` source to a local file. That file is then just text — safe to paste into a chat, attach, or commit somewhere, without giving the recipient any Claude Design account access at all.

**Prerequisite**: the `claude-design` MCP tools must be callable in this session (see `design-env-setup`) — this only works for projects the *current* Claude Design account can already read. If the person running this skill isn't a member/owner of the project, get them added first; this skill doesn't grant access, it only re-packages access someone already has.

## Step 1 — Get the target from the user

Ask for one of these — don't guess or invent one:

- A Claude Design screen **`open_url`** (the shareable link), or
- A **`project_id`** directly, optionally with a specific screen path if they already know it

If neither is given, ask explicitly which one they have. An `open_url` is usually easier for a non-technical requester to grab; a `project_id` is fine if they're already working in Claude Design/Claude Code.

## Step 2 — Resolve to `project_id` + path

- If given an `open_url`: parse the `project_id` out of it. If the URL's shape doesn't parse cleanly or the parsed value doesn't resolve, **don't guess a different one** — ask the user for the `project_id` directly instead.
- Confirm it resolves: `get_project(project_id)`.
- List the screens: `list_files(project_id)`.
  - If the user already named a specific path, confirm it's in that list.
  - If they didn't (or said "the whole project"), show the list and ask which screen(s) to export — or confirm "all of them" if that's what they asked for.

## Step 3 — Extract

For each requested screen:

```
read_file(project_id, path: "<screen>.dc.html")
```

This is the same call `spec-to-design`/`design-to-code` use to read a screen's real markup — nothing fancier is needed. Don't also pull the design-system bundle assets (`_ds_bundle.js`/`.css`) — the recipient's own project needs the design system installed as a real dependency anyway (`design-to-code` Step 0), not Claude Design's bundle copy; the screen's `.dc.html` alone already carries the component names and prop values `design-to-code` needs.

## Step 4 — Save locally

Write each screen's content to a local file named after the screen, e.g. `./<screen-name>.dc.html`, in the current working directory unless the user asked for somewhere else. For a multi-screen export, one file per screen — don't concatenate them into a single file, since `design-to-code` processes one screen at a time anyway.

## Step 5 — Report

Tell the user exactly which file(s) were written and their paths. Remind them the content is plain markup, safe to hand off directly (paste, attach, send) — and that they should send the **`.dc.html` file content itself**, never a `serve_url` (that's a bearer-token render link, not part of this export, and shouldn't be shared with anyone).
