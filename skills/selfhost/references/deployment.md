# qifa.yaml Examples

Annotated configurations following selfhost conventions. All hosts, domains, and
registries are placeholders. Check field names against the installed Qifa
version (`qifa config` prints the loaded config with defaults applied).

## Public web app (registry image, TLS, secrets)

The standard case. `prod-01/myapp/qifa.yaml`:

```yaml
service: myapp
image: registry.example.com/myapp:1.4.2   # pinned; CI publishes, this file selects

servers:
  web:
    hosts: [prod-01]                      # SSH alias

proxy_boot:
  hosts: [prod-01]                        # kamal-proxy host; `qifa proxy boot` once

proxy:
  host: app.example.com
  app_port: 3000
  ssl: true
  tls:
    source: qifa                          # issued with `qifa cert issue` (DNS-01)
    provider: cloudflare
  healthcheck:
    path: /up                             # traffic switches only after this passes

registry:
  server: registry.example.com
  username: deploy
  password_env: REGISTRY_PASSWORD         # read from the deployer's environment

env:
  clear:
    APP_ENV: production                   # non-secret only
  secret_command: sops --decrypt secrets.sops.env   # relative to this file

ssh:
  user: deploy
```

## Web + worker deployed together

One unit, two roles. Both roll out on every deploy. Split into two directories
only if they ship independently.

```yaml
service: myapp
image: registry.example.com/myapp:1.4.2

servers:
  web:
    hosts: [prod-01]
  worker:
    hosts: [prod-01]
    cmd: bin/worker                       # non-web roles are not proxied

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

env:
  secret_command: sops --decrypt secrets.sops.env
```

## Off-the-shelf image (no build)

Third-party software. Pin an exact version.

```yaml
service: gatus
image: ghcr.io/twin/gatus:v5.12.1

servers:
  web:
    hosts: [prod-01]
    volumes:
      - /srv/gatus/config:/config

proxy_boot:
  hosts: [prod-01]

proxy:
  host: status.example.com
  app_port: 8080
  ssl: true
  tls:
    source: qifa
    provider: cloudflare
  healthcheck:
    path: /health
```

## App with an accessory

Accessories run beside the app on the shared Docker network. The app reaches it
as `<service>-accessory-<name>`.

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

accessories:
  redis:
    image: redis:7.4-alpine
    host: prod-01
    volumes:
      - /srv/myapp/redis:/data

env:
  clear:
    REDIS_URL: redis://myapp-accessory-redis:6379
  secret_command: sops --decrypt secrets.sops.env
```

Accessory `env` is plaintext in Git. Don't use it for passwords; prefer images
that read credentials from a mounted file, or keep the service unauthenticated
on the private Docker network only.

## Internal-only service

No hostname, no certificate, no published port. Reachable only on the Docker
network as `<service>-<role>`.

```yaml
service: ocr
image: registry.example.com/ocr:2.0.1

servers:
  api:                                    # non-`web` role: not proxied
    hosts: [prod-01]
    app_port: 8080                        # no `port:` means nothing published on the host

proxy:
  healthcheck:
    path: /healthz
```

## Certificate commands

```bash
# issue before first deploy (extra hosts become SANs)
sops exec-env prod-01/cloudflare.sops.env \
  'qifa -c prod-01/myapp/qifa.yaml cert issue app.example.com --provider cloudflare --email ops@example.com'

# scheduled renewal for the whole server, then redeploy affected apps
sops exec-env prod-01/cloudflare.sops.env \
  'qifa -c prod-01/myapp/qifa.yaml cert renew --all --provider cloudflare --email ops@example.com'

qifa -c prod-01/myapp/qifa.yaml cert list --expiry
```
