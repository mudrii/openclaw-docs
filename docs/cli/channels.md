---
summary: "CLI reference for `openclaw channels` inspection and diagnostics"
read_when:
  - Inspecting configured channels
  - Checking channel capabilities or status
title: "`openclaw channels`"
---

# `openclaw channels`

Use `openclaw channels` to inspect messaging-channel accounts, status, logs,
and provider-specific capabilities.

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
openclaw channels resolve --channel slack --kind user alice
openclaw channels logs --channel all --lines 200
openclaw channels add --channel slack --account default
openclaw channels remove --channel slack --account default
openclaw channels login --channel imessage
openclaw channels logout --channel imessage
```

`channels list` shows chat channels only: configured accounts by default, with
`installed`, `configured`, and `enabled` status tags per account. Pass `--all`
to also surface bundled channels that have no configured account yet and
installable catalog channels that are not on disk.

Provider auth and usage snapshots no longer print from `channels list`. Use
`openclaw models auth list` for provider auth profiles, and use
`openclaw status` or `openclaw models list` for usage/model availability.

For Discord voice channels, the capabilities probe flags missing `ViewChannel`,
`Connect`, `Speak`, `SendMessages`, and `ReadMessageHistory` before `/vc join`.

Other stable subcommands cover setup and diagnostics:

- `channels resolve` resolves channel/user names to stable IDs.
- `channels logs` reads recent channel lines from the Gateway log file.
- `channels add` and `channels remove` add, update, disable, or delete channel account config.
- `channels login` and `channels logout` run channel-specific account-link flows when supported.
