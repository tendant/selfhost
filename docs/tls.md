# TLS

Public traffic is encrypted twice: browser to Cloudflare (Cloudflare's edge
certificate), and Cloudflare to the server (a Let's Encrypt certificate served by
kamal-proxy). Qifa issues and renews the origin certificate with lego, using the
Cloudflare DNS API for the DNS-01 challenge.

DNS-01 works behind Cloudflare's proxy, needs no inbound port 80, and works for
private hosts and wildcards.

## Cloudflare

| Setting | Value |
|---|---|
| DNS record | Proxied (orange cloud) |
| SSL/TLS mode | **Full (strict)** |
| Always Use HTTPS | On, where appropriate |
| API token | Permission `Zone → DNS → Edit`, restricted to the zones the server needs. One token per server (or environment), never an account-wide key. |

Store the token encrypted in the infrastructure repository:

```bash
# prod-01/cloudflare.sops.env (plaintext before encryption)
CF_DNS_API_TOKEN=...
```

Keep it separate from app `secrets.sops.env` files so it is never injected into
application containers.

## App configuration

```yaml
proxy:
  host: app.example.com
  ssl: true
  tls:
    source: qifa
    provider: cloudflare
    # staging: true   # use Let's Encrypt staging while testing
```

Qifa passes the certificate paths to kamal-proxy on every deploy. The certificate
must exist **before** the first deploy with `ssl: true`.

## Issue

```bash
cd infra/prod-01/myapp
sops exec-env ../cloudflare.sops.env \
  'qifa cert issue app.example.com --provider cloudflare --email ops@example.com'
```

- Extra hostnames after the first become SANs (for apps with `proxy.hosts`).
- Add `--staging` for test runs to avoid Let's Encrypt rate limits.
- Certificates are stored on the server in the kamal-proxy state volume. They
  survive `qifa proxy restart` and `qifa proxy upgrade`, but not
  `qifa proxy remove --purge`.

## Renew

Run renewal on a schedule (daily is fine; renewal only happens within 30 days of
expiry, tunable with `--renew-days`) from the operator machine or CI, once per
server:

```bash
sops exec-env prod-01/cloudflare.sops.env \
  'qifa -c prod-01/myapp/qifa.yaml cert renew --all --provider cloudflare --email ops@example.com'
```

Any `qifa.yaml` that targets the server works; `--all` renews every Qifa-managed
certificate on that server.

kamal-proxy reads a certificate only when a route is deployed and never re-reads
the file. When a certificate changed, `qifa cert renew` therefore re-registers
every route that uses it, replaying the exact command `qifa deploy` recorded on
the server. This swaps the route in place with no dropped requests; a
certificate that fails to load leaves the old route serving. A run where nothing
was due touches nothing, so run it daily.

Requires Qifa `16685e1` or later, and each app deployed once with it (routes
are recorded on deploy). Renewal reports any certificate whose app still needs
that first deploy.

A plain cron entry on the machine that holds the age key is enough:

```cron
15 2 * * * cd /path/to/infra && sops exec-env prod-01/cloudflare.sops.env 'qifa -c prod-01/myapp/qifa.yaml cert renew --all --provider cloudflare --email ops@example.com'
```

## Monitor

```bash
qifa -c prod-01/myapp/qifa.yaml cert list --expiry
```

Alert if any certificate is within 14 days of expiry: that means scheduled
renewal is failing. See Qifa's `docs/troubleshooting.md` for lego and DNS-01
failures.

## Alternatives

| `tls.source` | When |
|---|---|
| `qifa` (default) | Everything behind Cloudflare. |
| `kamal` | kamal-proxy autocert via HTTP-01. Only for hostnames that are **not** proxied by Cloudflare and have port 80 open. Renews without redeploys. |
| `static` | Bring-your-own certificate (e.g. Cloudflare Origin CA, corporate CA). You own renewal; set `cert_path` and `key_path`. |

Never commit TLS private keys or Cloudflare tokens to a public repository.
