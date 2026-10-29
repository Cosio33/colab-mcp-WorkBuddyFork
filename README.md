# Colab-mcp

An MCP server for bridging your local agent to a Colab session in the browser.

# Supported Clients
This MCP server requires client support for `notifications/tools/list_changed` and for the client to be running locally on your device. 

Popular clients that fit these criteria include:
- Gemini CLI
- Claude Code
- Windsurf


# Setup

- Install `uv` (`pip install uv`)
- Configure for usage (eg for mcp.json style services):

```
...
  "mcpServers": {
    "colab-mcp": {
      "command": "uvx",
      "args": ["git+https://github.com/googlecolab/colab-mcp"],
      "timeout": 30000
    }
  }
...
```

(If you have a non-standard default package index (**Googlers**), you may also need to add `--index https://pypi.org/simple`)

# Local Setup Notes (WorkBuddy AI)

> **This section is specific to WorkBuddy AI.** WorkBuddy reads MCP servers from its own
> `mcp.json` and supports per-server `env` and `timeout` fields, which is what the pinned
> port/token configuration relies on. Other clients (Gemini CLI, Claude Code, Windsurf) do not
> need any of this: use the standard setup above and ignore this section.

This clone carries a local patch plus a pinned launcher configuration. Paths and the token are
written as placeholders on purpose, so this document can be committed without leaking anything
machine-specific. With the new environment variables absent, the server behaves exactly like
upstream.

_Last verified: 2026-10-29._

## Why the patch exists

The proxy WebSocket server used to pick a random port (`port=0`) and a random token
(`secrets.token_urlsafe(16)`) on every start. The Colab frontend connects using both values,
read from the page URL fragment:

```
https://colab.research.google.com/notebooks/empty.ipynb#mcpProxyToken=<token>&mcpProxyPort=<port>
```

The MCP client does not keep the stdio server alive between turns; it relaunches it on demand
(observed: ~1 relaunch per turn, 15 launches in ~20 minutes). Each relaunch produced a new port
and token, so the open Colab tab ended up pointing at a dead port and the notebook tools
silently disappeared. Symptoms: notebook tools vanish after a pause, and in the worst case
`MCP initialize timed out`.

## The patch

`src/colab_mcp/websocket_server.py` (+6/−3):

```diff
+import os
...
-        self.port = 0
+        self.port = int(os.environ.get("COLAB_MCP_PORT") or 0)
...
-        self.token = secrets.token_urlsafe(16)
+        self.token = os.environ.get("COLAB_MCP_TOKEN") or secrets.token_urlsafe(16)
...
-            port=0,
+            port=self.port,
```

Both variables are optional. Unset or `0` keeps the upstream behaviour (ephemeral port, random
token), so this change is backward compatible.

## Environment variables

| Variable | Purpose | Value used here |
|---|---|---|
| `COLAB_MCP_PORT` | Fixed proxy WebSocket port. Unset/`0` = ephemeral. | `51765` |
| `COLAB_MCP_TOKEN` | Fixed bearer token for the proxy. Unset = random per start. | `<TOKEN>` (generate once, keep private) |

## Client configuration

WorkBuddy AI, in `%USERPROFILE%\.workbuddy-ai\mcp.json`:

```json
"colab-mcp": {
  "command": "<REPO>\\.venv\\Scripts\\python.exe",
  "args": ["-c", "from colab_mcp import main; main()"],
  "env": {
    "COLAB_MCP_PORT": "51765",
    "COLAB_MCP_TOKEN": "<TOKEN>"
  },
  "timeout": 60000
}
```

Replace the placeholders:

| Placeholder | What to put |
|---|---|
| `<REPO>` | Absolute path of this clone, e.g. `C:\\path\\to\\colab-mcp` |
| `<TOKEN>` | A value from `python -c "import secrets; print(secrets.token_urlsafe(16))"` |

On macOS/Linux the interpreter is `<REPO>/.venv/bin/python` and the config file is
`~/.workbuddy-ai/mcp.json`.

Why it is launched this way:

- **Not `.venv\Scripts\colab-mcp.exe`.** That file is a uv-generated trampoline shim. On Windows
  it can fail to open its target (`Failed to open executable ... permission denied,
  os error 5`), so the process dies before answering the handshake and the client reports
  `MCP initialize timed out`. Calling the venv interpreter directly avoids the shim.
- **No `cwd` needed.** The package is installed editable
  (`_editable_impl_colab_mcp.pth`), so `from colab_mcp import main` resolves from any directory.
- **`uv run --directory <repo> colab-mcp` also works**, but it requires `uv.exe` on the client's
  PATH and re-resolves the project on every launch.

## Stable browser URL

Because port and token are pinned, this URL can be bookmarked and simply reloaded to reattach:

```
https://colab.research.google.com/notebooks/empty.ipynb#mcpProxyToken=<TOKEN>&mcpProxyPort=51765
```

## Workflow

1. Trigger the MCP from the client (any call, e.g. `open_colab_browser_connection`). The stdio
   process starts and listens on port 51765.
2. Open or reload the stable URL above. The Colab frontend attaches to the local WebSocket.
3. `notifications/tools/list_changed` fires and the notebook tools become available.

## Gotchas

- **The process must be alive for the port to listen.** Once the client releases the stdio
  server, nothing is listening and reloading the tab will not connect. Trigger the MCP first.
- **Single client only.** A second frontend connection is rejected with code `1013`
  ("Server is busy"). Close the previous Colab tab before attaching another.
- **A fixed port fails loudly.** If 51765 is already in use, the server exits instead of falling
  back to an ephemeral port. This is intentional: a broken URL should be obvious, not silent.
- **Reinstalling wipes the patch.** `uv sync --reinstall` or a fresh clone drops the source
  change, since it is not upstream.
- **Token hygiene.** `COLAB_MCP_TOKEN` is a localhost-only bearer token. It belongs in the
  WorkBuddy client config, never in this repo: this document only shows `<TOKEN>` placeholders.
- **Scratch notebook.** The connected tab is `/notebooks/empty.ipynb`, which is not saved to
  Drive. Use _File → Save a copy in Drive_ to persist work.
- **Editing a Drive notebook.** The bridge has no Drive API access. To work on one, append the
  same `#mcpProxyToken=...&mcpProxyPort=...` fragment to that notebook's URL and keep a single
  tab attached.

# Issues & Discussions

We are using GitHub [discussions](https://github.com/googlecolab/colab-mcp/discussions) as the
place for issue discussion and feature requests. As discussions mature into action items, we
will add those items as issues. This helps us ensure that issues in the issue tracker are
well-understood, deduplicated, and actionable. For these reasons, **please do <u>NOT</u> open
issues directly.** 

# Contributing 
We unfortunately don't have the bandwidth to support review of external contributions, and we 
don't want user PRs to languish, so we aren't accepting any external contributions right now.

If you have a great idea or pain point, we would love to hear about it on our 
[discussions](https://github.com/googlecolab/colab-mcp/discussions) page - the preferred place 
for issue discussion and feature requests.

# Internal - For Colab Developers

### Prerequisites

- `uv` is required (`pip install uv`)
- Configure git hooks to run repo presubmits

```shell
git config core.hooksPath .githooks
```

### Gemini CLI setup

```
...
  "mcpServers": {
    "colab-mcp": {
      "command": "uv",
      "args": ["run", "colab-mcp"],
      "cwd": "/path/to/github/colab-mcp",
      "timeout": 30000
    }
  }
...
```
