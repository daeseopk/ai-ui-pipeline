# MCP raw-HTTP fallback recipe

Use this doc for **every** `claude-design` tool call in this skill (Steps 2–4) when `design-env-setup` Step 5 determined the session is in fallback mode — i.e. `ToolSearch query="select:mcp__claude-design__list_design_systems"` came up empty even though the server is registered and both consents (Step 1's `designOauth`, Step 4's `agent_design_projects`) are genuinely granted.

Don't guess argument names from memory or from this skill's prose descriptions of each tool. Fetch the real schema once per session and read field names off it — the same "verify, don't assume" rule this skill already applies to component props (`references/claude-design-recipe.md`) applies here too, and for the same reason: a remembered shape can drift from what the server actually expects.

## Step A — discover the real tool schemas (once per session)

```bash
python3 mcp_call.py list > /tmp/claude-design-tools.json
```

This returns all ~23 tools (`list_projects`, `get_project`, `create_project`, `list_design_systems`, `get_claude_design_prompt`, `list_files`, `read_file`, `copy_files`, `finalize_plan`, `write_files`, `delete_files`, `create_support_js`, `render_preview`, `list_comments`, `ack_comments`, `get_conversation`, `put_conversation`, `add_member`, `remove_member`, `update_member_role`, `list_members`, `update_sharing`, `read_design_skill`) with their full `inputSchema`. Read the schema for whichever tool you're about to call before constructing its `arguments` object — do this once and keep the file around for the rest of the session instead of re-fetching per call.

## Step B — the call helper

Save this as `mcp_call.py` next to wherever you're working (scratch dir, not a deliverable). It hides the `initialize` handshake and response parsing so every subsequent call is a one-liner.

```python
#!/usr/bin/env python3
"""Raw JSON-RPC bridge to the claude-design MCP server.
Only for use when ToolSearch confirms the native mcp__claude-design__* tools
are not loaded in this session (see design-env-setup Step 5)."""
import json
import os
import subprocess
import sys
import urllib.request

ENDPOINT = "https://api.anthropic.com/v1/design/mcp"


def _token():
    raw = subprocess.run(
        ["security", "find-generic-password", "-a", os.environ["USER"],
         "-s", "Claude Code-credentials", "-w"],
        capture_output=True, text=True, check=True,
    ).stdout
    return json.loads(raw)["claudeAiOauth"]["accessToken"]


def _parse(raw):
    # The server may frame the reply as plain JSON or as an SSE
    # `data: {...}` event depending on Accept negotiation — handle both.
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        for line in raw.splitlines():
            if line.startswith("data:"):
                return json.loads(line[len("data:"):].strip())
        raise


def _post(method, params, token, session_id):
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
        "MCP-Protocol-Version": "2025-06-18",
    }
    if session_id:
        headers["Mcp-Session-Id"] = session_id
    body = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params}).encode()
    req = urllib.request.Request(ENDPOINT, data=body, method="POST", headers=headers)
    with urllib.request.urlopen(req) as resp:
        new_session_id = resp.headers.get("Mcp-Session-Id", session_id)
        raw = resp.read().decode()
    return _parse(raw), new_session_id


def rpc(method, params):
    token = _token()
    _, session_id = _post(
        "initialize",
        {"protocolVersion": "2025-06-18", "capabilities": {},
         "clientInfo": {"name": "mcp_call-fallback", "version": "1.0.0"}},
        token, None,
    )
    result, _ = _post(method, params, token, session_id)
    return result


if __name__ == "__main__":
    if len(sys.argv) < 2 or sys.argv[1] not in ("list", "call"):
        print("usage: mcp_call.py list | mcp_call.py call <tool_name> ['<json_args>']", file=sys.stderr)
        sys.exit(1)
    if sys.argv[1] == "list":
        out = rpc("tools/list", {})
    else:
        if len(sys.argv) < 3:
            print("usage: mcp_call.py call <tool_name> ['<json_args>']", file=sys.stderr)
            sys.exit(1)
        tool_name = sys.argv[2]
        args = json.loads(sys.argv[3]) if len(sys.argv) > 3 else {}
        out = rpc("tools/call", {"name": tool_name, "arguments": args})
    print(json.dumps(out, indent=2, ensure_ascii=False))
```

Never print the token itself — this script only ever holds it in a local variable and puts it straight into a request header, and nothing in the recipe below asks you to echo it.

## Step C — calling it per step

Every native call in this skill becomes:

```bash
python3 mcp_call.py call <tool_name> '<json_args>'
```

where `<json_args>` is a JSON object built from the field names in `/tmp/claude-design-tools.json` (Step A), following the same conceptual shape this skill's steps already describe:

| Skill step | Tool name | Args come from |
|---|---|---|
| Step 2 | `list_design_systems` | no args |
| Step 2 | `get_claude_design_prompt` | `design_system_id` from the Step 2 `list_design_systems` result |
| Step 3 | `create_project` | `name`, `design_system_id` |
| Step 3 | `create_support_js` | `project_id` from the `create_project` result |
| Step 3 | `copy_files` | `project_id` + a `files` array, one entry per asset (`_ds_bundle.js`, `_ds_bundle.css`, `styles.css`) — check the schema for exact per-entry field names (this skill's prose uses `src`/`dest`/`src_project_id`, confirm against Step A's schema before trusting that from memory) |
| Step 4 | `write_files` | `project_id` + the screen's path/content — confirm field names via Step A, this skill doesn't hardcode them |
| Step 4 | `render_preview` | `project_id`, `path` |
| Step 4 (prop lookups) | `read_file` | `project_id` (the *design system's* id, not the content project's), `path` to the `.d.ts` |

If a call returns `{"error":"needs_consent", ...}` mid-task, **stop** — this means Step 4 of `design-env-setup` (Claude Design product access) has lapsed or was scoped narrower than what you're now doing (e.g. a per-project Revoke). Don't retry the same call hoping it resolves itself; tell the user and point them back to `claude.ai/design/settings`.

## Cost of this path

Each `rpc()` call does two round trips (`initialize` then the real method) because session continuity across separate `mcp_call.py` invocations isn't assumed. This is fine for the call volumes in this skill (dozens of calls per spec, not thousands) — don't optimize it away by hand-rolling a session-reuse layer for a one-off fallback path.
