# Agent Instructions

This is a **public** standards repository: documentation, templates, and an
agent skill for self-hosted infrastructure. It contains no production
configuration.

## Rules

- Never add credentials, API tokens, private keys, age identities, server
  inventories, IP addresses, or real private hostnames. Use `example.com`,
  `prod-01`, and `10.0.0.x` style placeholders.
- `docs/architecture.md` is the design source of truth. Keep other docs and
  the skill consistent with it; change the design doc first when a convention
  changes.
- Describe tool behavior only as the installed tool actually supports it.
  Check Qifa's `README.md` and `internal/config/config.go` before documenting
  a `qifa.yaml` field or command.
- Link to upstream tool docs instead of duplicating them.
- Keep docs short. No speculative features (see "Future Evolution" in the
  design doc).

## Layout

- `docs/`: standards, one topic per file
- `skills/selfhost/`: agent skill (`SKILL.md` + `references/`)
- `templates/`: files consuming repositories copy
