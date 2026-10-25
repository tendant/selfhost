# Infrastructure Instructions

Standards: https://github.com/tendant/selfhost (pinned: `v0.1.0`)

Follow the `selfhost` skill when available.

## Rules

- Organize configuration by server: `<server>/qifa.yaml`, or
  `<server>/<unit>/qifa.yaml` for multiple independently deployed units.
- Use Qifa as the deployment source of truth. Validate with `qifa config` and
  `qifa deploy --dry-run` before any deploy.
- Use SOPS + age for secrets. Commit only `*.sops.env` files; never write,
  print, or commit decrypted values.
- Use Kamal Proxy for ingress (`qifa proxy boot` once per server).
- Use Cloudflare DNS-01 for TLS: `proxy.tls.source: qifa`,
  `provider: cloudflare`. The Cloudflare token lives in
  `<server>/cloudflare.sops.env`.
- Make the smallest change; don't touch unrelated servers or units.
- No templates: every `qifa.yaml` holds literal values and is used as
  committed. No envsubst, render steps, generators, or Makefile wrappers.
  Repeated lines are fine.
- Never deploy, roll back, or change proxies or certificates in production
  without explicit authorization.

<!-- Add repository-specific notes below: servers, owners, deploy schedule. -->
