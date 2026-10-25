---
name: selfhost
description: >
  Infrastructure deployment practices for independent
  Docker servers using Qifa, Kamal Proxy, Cloudflare,
  Let's Encrypt, and SOPS. Use when designing,
  reviewing, or modifying self-hosted infrastructure.
---

# Selfhost

Conventions for infrastructure repositories that deploy Docker apps to
independent servers. Qifa deploys and manages certificates, kamal-proxy routes
traffic, Cloudflare fronts public traffic, Let's Encrypt issues certificates via
DNS-01, and SOPS + age encrypts secrets in Git.

Full standards: https://github.com/tendant/selfhost (`docs/`).

## Rules

1. **Inspect before changing.** Read the existing `qifa.yaml`, the server
   directory, `.sops.yaml`, and the repository `AGENTS.md` first. Match what is
   there.
2. **Server-first layout.** `<server>/qifa.yaml`, or
   `<server>/<unit>/qifa.yaml` when a server runs several independently
   deployed units. Don't split API/web/worker unless they deploy independently;
   use Qifa roles instead.
3. **No templates or extra layers.** Each `qifa.yaml` holds literal values
   and is used exactly as committed: no `envsubst`/`${VAR}` placeholders,
   render steps, `build/` directories, config generators, or Makefile
   wrappers. Repeated lines are acceptable; abstraction is not.
4. **Secrets stay encrypted.** Only `*.sops.env` files are committed. Never
   write, print, or commit decrypted values. App secrets reach containers via
   `env.secret_command: sops --decrypt secrets.sops.env`. Never put secrets in
   `env.clear` or accessory `env` (both are plaintext in Git).
5. **Servers are independent.** No cross-server dependencies, shared
   certificate storage, or central controllers.
6. **TLS is separate from app deploys.** Certificates are issued and renewed
   with `qifa cert`, using `proxy.tls.source: qifa` and
   `provider: cloudflare`. The Cloudflare token lives in
   `<server>/cloudflare.sops.env`, never in app secrets.
7. **Verify tool capabilities.** Check the installed Qifa's README or
   `qifa config` output before using a field or command. Don't invent
   `qifa.yaml` keys.
8. **Smallest change.** Usually one `qifa.yaml`. Never touch unrelated units or
   servers.
9. **Validate before deploying.** `qifa config`, then
   `qifa deploy --dry-run`.
10. **Production needs explicit authorization.** Do not run `qifa deploy`,
    `rollback`, `remove`, `proxy` commands, or `cert` commands against
    production without the user's explicit go-ahead for that specific action.

## Workflow

### Change an existing deployment

1. Read `<server>/<unit>/qifa.yaml`.
2. Make the minimal edit (often only the `image:` tag).
3. `qifa -c <path>/qifa.yaml config` to validate.
4. `qifa -c <path>/qifa.yaml deploy --dry-run` and show the user the plan.
5. Commit. Deploy only after authorization; Qifa refuses dirty trees.
6. Confirm with `qifa status`; on failure, propose `qifa rollback`.

### Add a new app

1. Pick the server; create `<server>/<service>/` if the server already hosts
   other units (move a single existing unit into its own subdirectory only if
   the user agrees).
2. Write `qifa.yaml` from [references/deployment.md](references/deployment.md).
   Pin the image to a version tag or digest.
3. Create secrets with `sops edit <dir>/secrets.sops.env` (the user types the
   values; never ask them to paste secrets into the chat).
4. Cloudflare: proxied DNS record, SSL mode Full (strict).
5. Issue the certificate before the first deploy:
   `sops exec-env <server>/cloudflare.sops.env 'qifa -c <dir>/qifa.yaml cert issue <host> --provider cloudflare --email <ops-email>'`
6. Validate, then deploy with authorization.

### Add a new server

1. Create `<server>/`, add a `.sops.yaml` rule for it.
2. `qifa doctor`, then `qifa proxy boot` once (with authorization).
3. Add `<server>/cloudflare.sops.env` with a zone-scoped `CF_DNS_API_TOKEN`.
4. Schedule a daily `qifa -c <server>/<unit>/qifa.yaml cert renew` per app
   (one plain cron line each, not `--all`). When the certificate changed it
   re-registers that app's route from its `qifa.yaml`, with no downtime.
   Never restart kamal-proxy to pick up certificates.

## Review checklist

- [ ] No plaintext secrets, tokens, or keys anywhere in the diff
- [ ] Image pinned (tag or digest), not `:latest` in production
- [ ] `proxy.healthcheck.path` set for proxied roles
- [ ] `ssl: true` comes with `tls.source: qifa` and an issued certificate
- [ ] Change limited to the intended unit and server
- [ ] `qifa config` passes
- [ ] No new abstraction (templates, scripts, shared config) without a stated
      need

## References

- [references/deployment.md](references/deployment.md): annotated
  `qifa.yaml` examples
- Standards docs: `docs/deployment.md`, `docs/tls.md`, `docs/secrets.md`,
  `docs/repository-layout.md` in the selfhost repository
- Qifa: https://github.com/tendant/qifa
