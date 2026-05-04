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

The plugin uses a default-deny per-node path policy. Operators approve paths under:

```json5
{
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

## Limits

File-transfer round trips are capped at 16 MB. Larger payloads should use a purpose-built sync, archive, or storage workflow rather than agent tool transport.
