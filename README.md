# AgentMemory Offline

Persistent, project-aware memory for local coding agents, derived from
[rohitg00/agentmemory](https://github.com/rohitg00/agentmemory).

This fork keeps memory on the machine and runs its memory services without
outbound network access. It preserves the useful parts of AgentMemory—session
capture, recall, summaries, semantic structure, MCP tools, hooks, and a local
viewer—while disabling cloud model providers, telemetry, runtime downloads,
and update checks.

## What “offline” means

The runtime is designed so that memory content is never sent to a remote
service:

- no cloud LLM provider is selected;
- no provider API key is required;
- no telemetry or update check is enabled;
- MCP points to the locally installed executable, never to `npx`;
- on macOS, the bundled engine is launched through `sandbox-exec` with
  non-loopback outbound traffic denied;
- local retrieval falls back to lexical/BM25 and the local graph.

Building or installing dependencies may still require access to npm and GitHub.
That is outside the runtime guarantee. Linux and Windows do not currently get
the macOS process-level network sandbox; use an OS firewall or container
network policy there if you need an independently enforced boundary.

See [docs/OFFLINE_SECURITY.md](docs/OFFLINE_SECURITY.md) for the threat model,
ports, limitations, and verification procedure.

## Where memory is stored

The database and engine state live outside project repositories by default:

| Platform | Default location |
| --- | --- |
| macOS | `~/Library/Application Support/agentmemory` |
| Linux | `$XDG_DATA_HOME/agentmemory` or `~/.local/share/agentmemory` |
| Windows | `%APPDATA%\agentmemory` |

Override the location with `--data-dir <path>` or
`AGENTMEMORY_DATA_DIR=<path>`. Memory records carry project/session context, so
multiple sessions for the same project can contribute to the same project
memory without mixing their session history.

## Architecture

```text
Local agent
  ├─ lifecycle hooks ───────────────┐
  └─ MCP tools ────────────────┐    │
                               v    v
                     AgentMemory REST service :3111
                         ├─ local retrieval
                         ├─ local graph/index
                         ├─ session observations
                         └─ local viewer :3113
                                      │
                                      v
                              local iii engine
```

The default interfaces are local HTTP/MCP surfaces. Set
`AGENTMEMORY_SECRET` to require bearer authentication even on localhost.

## Build and run

Requirements: Node.js 20 or newer and the pinned iii engine version shown in
`src/version.ts`.

```bash
npm install
npm run build
npm link

agentmemory --data-dir "$HOME/.local/share/agentmemory"
```

The application will not download the engine at runtime in offline mode. Install
the pinned engine ahead of time while online, or place the compatible binary in
the private AgentMemory bin directory.

For MCP-only clients:

```json
{
  "mcpServers": {
    "agentmemory": {
      "command": "agentmemory",
      "args": ["mcp"],
      "env": {
        "AGENTMEMORY_URL": "http://127.0.0.1:3111"
      }
    }
  }
}
```

## Host integration

The repository includes portable skills and lifecycle hooks. Installation must
merge these declarations with the host's existing configuration; it must not
replace unrelated hooks or skills. The MCP manifest calls the locally installed
`agentmemory` executable.

### Native Codex Desktop plugin

The offline fork now includes a native package at
`plugins/agentmemory-offline` and a repository-local marketplace manifest at
`.agents/plugins/marketplace.json`.

Install it from a local clone:

```bash
codex plugin marketplace add /absolute/path/to/agentmemory-offline
codex plugin add agentmemory-offline@offline-memory
```

The native package registers:

- 15 project-memory skills;
- six lifecycle hooks (`SessionStart`, `UserPromptSubmit`, `PreToolUse`,
  `PostToolUse`, `PreCompact`, and `Stop`);
- a self-contained MCP bridge resolved from the plugin cache;
- a fixed loopback endpoint at `http://127.0.0.1:3111`.

The package does not accept a remote memory URL. If the full local service is
not running, its MCP bridge retains seven essential operations in
`~/.agentmemory/standalone.json`. When the service is running, memories use its
configured local data directory and remain grouped by repository and session.

Restart Codex Desktop after installation so the newly registered skills, MCP
server, and lifecycle hooks are discovered.

## Current limitations

- Runtime network enforcement is strongest on macOS.
- The iii engine may create an internal listener on port `49134`. The macOS
  sandbox rejects non-loopback inbound traffic, but users on other systems must
  bind or firewall that port themselves.
- Optional vector embeddings are not part of the guaranteed offline profile.
  BM25 and graph retrieval remain available.
- Mesh/peer synchronization is outside the offline profile and must remain
  disabled.

## Verification

```bash
npm test
npm run build
rg -n "OPENAI_API_KEY|ANTHROPIC_API_KEY|OPENROUTER_API_KEY" .env 2>/dev/null
lsof -nP -iTCP -sTCP:LISTEN | rg "3111|3112|3113|49134"
```

For a stronger test, deny all external traffic at the OS firewall, start the
service, create a memory, stop it, start it again, and recall the same memory.

## Provenance and license

This is an independent offline adaptation of AgentMemory. The original project
and attribution are preserved in [docs/UPSTREAM_README.md](docs/UPSTREAM_README.md).
Licensed under Apache-2.0; see [LICENSE](LICENSE).
