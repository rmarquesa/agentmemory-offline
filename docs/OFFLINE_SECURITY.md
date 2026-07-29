# Offline security model

## Protected assets

The protected assets are prompts, tool observations, repository paths, commit
context, summaries, memories, and retrieval results.

## Runtime trust boundary

Trusted components are the local host application, this repository's hooks and
MCP process, the AgentMemory service, the pinned iii engine, and the selected
local data directory. Package registries, update endpoints, model providers,
remote MCP servers, mesh peers, and telemetry collectors are outside the
boundary.

## Controls

1. Provider selection is forced to local/no-op behavior for the offline profile.
2. Engine update checks and telemetry are disabled through arguments and
   environment variables.
3. Runtime package resolution through `npx` is not used.
4. The macOS engine process runs under `offline.sb`, which denies non-loopback
   outbound traffic and non-loopback inbound traffic.
5. MCP and hook URLs use `127.0.0.1` or `localhost`.
6. Memory storage is local and configurable with `AGENTMEMORY_DATA_DIR`.

## Local ports

| Port | Purpose | Expected exposure |
| --- | --- | --- |
| 3111 | REST API | loopback |
| 3112 | event stream | loopback |
| 3113 | viewer | loopback |
| 49134 | iii engine transport | internal; firewall on non-macOS |

Do not expose these ports through router forwarding, reverse proxies, tunnels,
or container publishing. Set `AGENTMEMORY_SECRET` when other local users or
processes are not fully trusted.

## Explicitly unsupported in the offline profile

- cloud summarization and embeddings;
- mesh synchronization;
- remote MCP servers;
- telemetry;
- automatic engine/package downloads;
- automatic update checks.

## Audit checklist

```bash
# Inspect compiled/runtime references before release.
rg -n "https://|wss://" src plugin iii-config.yaml offline.sb

# Confirm listeners after startup.
lsof -nP -iTCP -sTCP:LISTEN | rg "3111|3112|3113|49134"

# Confirm no established external connections owned by the processes.
lsof -nP -iTCP -sTCP:ESTABLISHED | rg "agentmemory|iii"

# Run with the network disconnected and verify persistence.
agentmemory --data-dir /path/to/test-data
```

Source-code links and documentation links may contain remote URLs without being
runtime connections. Every executable code path found by the first command must
be classified before a release.

## Residual risks

Dependencies remain part of the software supply chain. Pin and audit them before
building a trusted release. A compromised host process with filesystem access
can read the local database. Disk encryption, strict directory permissions, and
an OS firewall remain recommended.
