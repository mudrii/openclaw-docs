---
summary: "Fetch, list, and write files on paired nodes with the file-transfer plugin"
read_when:
  - You are configuring paired-node file movement
  - You need binary-safe file transfer without shell stdout truncation
title: "File Transfer"
---

# File Transfer

The bundled `file-transfer` plugin adds dedicated paired-node file operations for agent tools:

- `file_fetch`
- `dir_list`
- `dir_fetch`
- `file_write`

Use it when an agent needs binary-safe file movement on a paired node and shell output limits would make `exec` unreliable.

## Distribution

- Package: `@openclaw/file-transfer`
- Install route: included with OpenClaw
- Surface: plugin tools

## Safety Model

The plugin uses two operator-controlled gates:

- `gateway.nodes.allowCommands` must allow the paired-node command name: `file.fetch`, `dir.list`, `dir.fetch`, or `file.write`.
- `plugins.entries.file-transfer.config.nodes.<node>` must allow the requested path through `allowReadPaths` or `allowWritePaths`.

Operators approve paths under:

```json5
{
  gateway: {
    nodes: {
      allowCommands: ["file.fetch", "dir.list", "dir.fetch", "file.write"],
    },
  },
  plugins: {
    entries: {
      "file-transfer": {
        enabled: true,
        config: {
          nodes: {
            "node-id": {
              ask: "on-miss",
              allowReadPaths: ["/approved/path", "/approved/path/**"],
              allowWritePaths: ["/approved/output", "/approved/output/**"],
              denyPaths: ["/approved/path/secrets/**"],
              maxBytes: 16777216,
              followSymlinks: false,
            },
          },
        },
      },
    },
  },
}
```

Symlink traversal is refused by default. Enable `followSymlinks` only for paths where the target traversal behavior is intentional and reviewed.

`ask` controls approval prompts after the static policy check:

- `off` or unset: deny misses silently.
- `on-miss`: allow configured paths silently, ask only when no allow pattern matched.
- `always`: ask on every call, even when an allow pattern matched.

`denyPaths` always wins, including when `ask` is `always`.

## Limits

| Tool | Default | Hard ceiling |
| ---- | ------- | ------------ |
| `file_fetch` | 8 MB | 16 MB |
| `dir_fetch` | 8 MB compressed archive | 16 MB compressed archive |
| `file_write` | N/A | 16 MB source payload |
| `dir_list` | 200 entries per page | 5000 entries per page |

Larger payloads should use a purpose-built sync, archive, or storage workflow rather than agent tool transport. Generic `nodes.invoke` rejects `file.fetch`, `dir.list`, `dir.fetch`, and `file.write`; use the dedicated file-transfer tools so path policy, audit logging, size caps, and symlink checks always apply.
