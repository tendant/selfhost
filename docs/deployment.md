# Deployment

[Qifa](https://github.com/tendant/qifa) is the deployment tool and the source of
truth for how a unit runs. This page covers conventions; Qifa's README covers the
commands and config schema.

## Flow

```text
App repo ──CI──> image in registry ──> infra repo (qifa.yaml pins image)
                                            │
                                        qifa deploy  (operator machine or CI)
                                            │ SSH
                                       target server
                                            │
                              kamal-proxy ──> app containers
```

- Application source code stays in its own repository. CI (GitHub Actions or
  Gitea Actions) builds and pushes the image.
- The infrastructure repository owns `qifa.yaml` and the desired image version.
- Qifa runs from an operator machine or CI, SSHing to the target as one shared
  `deploy` user. Deployment state lives on the servers (container labels and the
  deploy lock), never in Git.

## Requirements

| Requirement | How |
|---|---|
| Immutable images | Pin `image:` to a version tag or `@sha256:` digest. Qifa resolves tags to digests at deploy time, but the config should still say exactly what runs. |
| Validate first | `qifa config` (parse and validate), then `qifa deploy --dry-run`. Run `qifa doctor` on new or changed servers. |
| Health before traffic | Set `proxy.healthcheck.path`. Qifa only switches traffic after the new container passes. |
| Rollback | `qifa rollback [version]`. Keep `prune.retain_containers` above zero so a previous version exists. |
| Isolation | Deploy one unit at a time with `-c <server>/<unit>/qifa.yaml`. Never batch unrelated units. |
| Secrets | `env.secret_command: sops --decrypt secrets.sops.env`. See [secrets.md](secrets.md). |
| No drift | `qifa env diff` before deploying, `qifa reconcile --dry-run` to check a server matches its config. |

## Standard `qifa.yaml` shape

```yaml
service: myapp
image: registry.example.com/myapp:1.4.2

servers:
  web:
    hosts: [prod-01]

proxy_boot:
  hosts: [prod-01]

proxy:
  host: app.example.com
  app_port: 3000
  ssl: true
  tls:
    source: qifa
    provider: cloudflare
  healthcheck:
    path: /up

registry:
  server: registry.example.com
  username: deploy
  password_env: REGISTRY_PASSWORD

env:
  clear:
    APP_ENV: production
  secret_command: sops --decrypt secrets.sops.env

ssh:
  user: deploy
```

More examples: [skills/selfhost/references/deployment.md](../skills/selfhost/references/deployment.md).

## New server

1. Provision the host with Docker and a `deploy` SSH user. Add an SSH alias
   matching the server directory name.
2. `qifa doctor` to check SSH, Docker, registry access, DNS, and clock.
3. `qifa proxy boot` once per host (idempotent). Upgrade later with
   `qifa proxy upgrade`. App deploys never change proxy boot config.
4. Create the Cloudflare DNS record and issue certificates ([tls.md](tls.md)).
5. `qifa deploy`.

## Changing a deployment

1. Edit the one `qifa.yaml` (usually just the image version).
2. `qifa config` and `qifa deploy --dry-run`.
3. Commit, then deploy from a clean tree. Qifa refuses a dirty tree unless
   `--allow-dirty` is passed; don't pass it for production.
4. Verify with `qifa status` and the app's health endpoint.
5. If it misbehaves: `qifa rollback`.

Production deploys require explicit human authorization.
