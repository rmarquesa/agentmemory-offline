# AgentMemory Offline plugin

This package is the native Desktop integration for the offline AgentMemory
fork. It bundles the skills, lifecycle hooks, and MCP bridge needed by the
host. It never uses a hosted memory API.

## Local-only boundary

- MCP and hooks use the fixed loopback endpoint `http://127.0.0.1:3111`.
- The package does not accept an environment override for a remote memory URL.
- The MCP bridge falls back to a reduced local store if the full local service
  is unavailable.
- No updater, telemetry client, remote MCP server, cloud backup, or
  synchronization feature is included in this package.

Repository and documentation links are metadata only. Installing or running
the plugin does not fetch them.

## Where memory lives

Persistent state remains on the local machine:

- `~/.agentmemory/standalone.json` — reduced MCP fallback store.
- `~/.agentmemory/` — local service state and configuration.
- Project/session records are keyed by the repository root and session
  identifier, so separate conversations for the same repository share project
  memory without merging unrelated projects.

The package itself is copied into the Desktop plugin cache. Removing or
updating the plugin does not delete `~/.agentmemory`.

## Components

- `skills/` — recall, remember, history, handoff, commit context, configuration,
  architecture, hooks, and API guidance.
- `hooks/hooks.json` — six lifecycle integrations.
- `scripts/` — loopback-only hook executables.
- `.mcp.json` and `runtime/standalone.mjs` — local MCP bridge.

The full 53-tool surface requires the local AgentMemory service on port 3111.
When it is unavailable, the bundled bridge preserves seven essential memory
operations in `standalone.json`.
