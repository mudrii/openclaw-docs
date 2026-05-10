---
summary: "CLI reference for model and provider auth inspection"
read_when:
  - Inspecting saved provider auth profiles
  - Checking model availability from the CLI
title: "`openclaw models`"
---

# `openclaw models`

Use `openclaw models` to inspect model availability and provider authentication.

```bash
openclaw models list
openclaw models list --provider openai
openclaw models auth list
openclaw models auth list --provider openai-codex
openclaw models auth list --json
```

`openclaw models auth list` is the provider-auth inspection path. Channel
listing is intentionally channel-only as of `v2026.5.7`; provider auth and usage
belong under `models auth list`, `models list`, and `status`.

