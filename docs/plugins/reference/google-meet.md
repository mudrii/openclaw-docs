---
summary: "Join Google Meet calls through Chrome or Twilio transports."
read_when:
  - You are installing, configuring, or auditing the google-meet plugin
title: "Google Meet plugin"
---

# Google Meet plugin

Join Google Meet calls through Chrome or Twilio transports.

The `google-meet` plugin contributes the `googlemeet` CLI command and the
`google_meet` agent tool. It supports `chrome`, `chrome-node`, and `twilio`
transports, and `agent`, `bidi`, and `transcribe` modes. `realtime` remains a
compatibility alias for `agent`. Twilio dial-in joins route through the Voice
Call/realtime bridge, and `realtime.introMessage` may be set to an empty string
for silent Chrome joins.

## Distribution

- Package: `@openclaw/google-meet`
- Install route: npm; ClawHub

## Surface

contracts: tools, CLI

## Related docs

- [google-meet](/plugins/google-meet)
