# Selfhost

**Opinionated, agent-friendly infrastructure practices for self-hosting applications.**

Selfhost is a collection of reusable infrastructure standards, deployment patterns, and AI agent skills for managing independent servers with Docker, Qifa, Kamal Proxy, Cloudflare, Let's Encrypt, and SOPS.

This document is the overall design. Operational detail lives in the topic docs: [repository layout](repository-layout.md), [deployment](deployment.md), [TLS](tls.md), and [secrets](secrets.md).

The goal is simple: **make self-hosted infrastructure easy to deploy, secure, maintain, and automate without unnecessary complexity.**

## 1. Design Principles

1. **Keep it simple.** Prefer existing tools over custom orchestration.
2. **Server-first organization.** Group configurations by target server.
3. **Deployment-unit ownership.** One configuration directory per independently deployed unit.
4. **Git as source of truth.** Version desired infrastructure configuration.
5. **Independent servers.** Avoid unnecessary cross-server dependencies.
6. **Secure by default.** Encrypt secrets and automate TLS.
7. **Agent-friendly.** Document conventions so AI coding agents can work consistently.
8. **No templates, minimal abstraction.** Configuration files hold literal values and are used as committed. Prefer repeated lines over layers that generate them.

## 2. Technology Stack

| Component | Technology | Responsibility |
|---|---|---|
| Deployment | Qifa | Docker deployments, health checks, traffic switching, rollback |
| Container runtime | Docker | Run application containers |
| Reverse proxy | Kamal Proxy | HTTP routing and traffic switching |
| DNS / CDN | Cloudflare | DNS, proxy, edge TLS, WAF |
| TLS | Let's Encrypt | Publicly trusted certificates |
| ACME client | Qifa (`qifa cert`, lego) | DNS-01 issuance and scheduled renewal |
| Secrets | SOPS + age | Encrypted configuration stored in Git |
| CI/CD | GitHub Actions / Gitea Actions | Build and publish images |

## 3. Architecture

```text
                     Cloudflare
                  DNS + HTTPS Proxy
                         |
             +-----------+-----------+
             |                       |
          Server A                Server B
             |                       |
         Kamal Proxy             Kamal Proxy
             |                       |
        Docker Apps              Docker Apps

                Qifa (operator machine / CI)
        deploys over SSH; issues and renews certificates
        via Let's Encrypt DNS-01, stored on each server
```

Each server has its own deployment and certificate state. Qifa holds no authoritative state on the operator machine.

No Kubernetes, central deployment controller, or shared certificate storage is required.

## 4. Repository Organization

This public repository stores reusable standards and agent skills, not production server configuration.

```text
selfhost/
├── README.md
├── AGENTS.md
│
├── docs/
│   ├── architecture.md
│   ├── repository-layout.md
│   ├── deployment.md
│   ├── tls.md
│   └── secrets.md
│
├── skills/
│   └── selfhost/
│       ├── SKILL.md
│       └── references/
│           └── deployment.md
│
└── templates/
    └── AGENTS.md
```

### Private infrastructure repositories

Actual deployment configuration belongs in a separate private repository.

Example:

```text
infra/
├── AGENTS.md
│
├── prod-01/
│   ├── qifa.yaml
│   └── secrets.sops.env
│
├── prod-02/
│   ├── qifa.yaml
│   └── secrets.sops.env
│
└── dev-01/
    ├── qifa.yaml
    └── secrets.sops.env
```

If a server contains multiple independently deployed applications, introduce subdirectories:

```text
prod-01/
├── project-a/
│   ├── qifa.yaml
│   └── secrets.sops.env
└── project-b/
    ├── qifa.yaml
    └── secrets.sops.env
```

Do not split API, web, and worker components unless they are independently deployed.

Use the configuration filenames and schema supported by the installed Qifa version.

## 5. Deployment Standards

Application source code stays in its own Git repository.

The infrastructure repository owns deployment configuration and desired image versions.

```text
Application Repository
         |
      CI Build
         |
    Docker Registry
         |
Infrastructure Repository
         |
        Qifa
         |
     Target Server
         |
     Kamal Proxy
         |
   Docker Containers
```

Deployment requirements:

- Prefer immutable image tags or digests.
- Validate configuration before deployment.
- Check application health before switching traffic.
- Preserve rollback capability.
- Do not redeploy unrelated applications.
- Keep deployment state separate from desired configuration.
- Reuse Qifa capabilities rather than implementing duplicate deployment logic.

## 6. TLS Management

Use Let's Encrypt with Cloudflare DNS-01.

Cloudflare configuration:

- Proxy enabled (orange cloud).
- SSL/TLS mode: Full (strict).
- HTTPS redirects enabled where appropriate.
- API tokens scoped to necessary DNS zones.

Use Qifa's built-in certificate management (`proxy.tls.source: qifa`):

1. `qifa cert issue` obtains a certificate via the Cloudflare DNS API before an app's first deploy.
2. Certificates are stored on each server in the kamal-proxy state volume.
3. A daily scheduled `qifa cert renew`, run per app from its `qifa.yaml`, renews certificates nearing expiry.
4. When a certificate changed, the same command re-registers the app's route, rebuilt from its `qifa.yaml`, since Kamal Proxy only reads certificates when a route is deployed. No downtime, no proxy restart.

Renewal never redeploys applications. Keep certificate logic in Qifa rather than building a parallel ACME setup or wrapper scripts.

HTTP-01 (`source: kamal`, kamal-proxy autocert) remains an alternative for individual hostnames but is not the preferred default. Bring-your-own certificates use `source: static`.

See [tls.md](tls.md) for the full procedure.

## 7. Secrets Management

Use SOPS with age encryption.

Rules:

- Store encrypted `*.sops.env` files in Git.
- Prefer self-contained files: each `*.sops.env` carries its own age recipients, so decrypting, editing and changing recipients need no repository-level `.sops.yaml`. A `.sops.yaml` remains optional.
- Never commit plaintext `.env` files.
- Use encrypted files as the source of truth.
- Restrict decryption identities.
- Avoid exposing secrets in logs.
- Keep Cloudflare API tokens and TLS private keys out of public repositories.
- Document secret rotation and recovery procedures.

A public standards repository must never contain actual credentials, production inventories, private hostnames, or sensitive operational details.

## 8. Reusable AI Agent Skill

The repository provides a reusable skill at:

```text
skills/selfhost/SKILL.md
```

Suggested skill definition:

```yaml
---
name: selfhost
description: >
  Infrastructure deployment practices for independent
  Docker servers using Qifa, Kamal Proxy, Cloudflare,
  Let's Encrypt, and SOPS. Use when designing,
  reviewing, or modifying self-hosted infrastructure.
---
```

The skill should instruct agents to:

1. Inspect existing Qifa configuration.
2. Follow server-first repository conventions.
3. Avoid duplicate configuration.
4. Keep secrets encrypted.
5. Preserve server independence.
6. Separate TLS lifecycle from application deployment.
7. Verify actual tool capabilities.
8. Make the smallest necessary change.
9. Validate before deployment.
10. Require explicit authorization for production changes.

## 9. Integrating with Other Repositories

Each infrastructure repository should contain a short `AGENTS.md` referencing this project (copy [templates/AGENTS.md](../templates/AGENTS.md)):

```markdown
# Infrastructure Instructions

Standards:
https://github.com/tendant/selfhost

Follow the selfhost skill when available.

Rules:
- Organize configuration by server.
- Use Qifa as the deployment source of truth.
- Use SOPS + age for secrets.
- Use Kamal Proxy for ingress.
- Use Cloudflare DNS-01 for TLS.
- Avoid unnecessary abstractions.
- Never deploy production changes without authorization.
```

Keep essential rules in the local `AGENTS.md`. A remote URL alone does not guarantee an agent will fetch or follow the standards.

For reproducibility, pin consuming projects to a known standards version or commit.

## 10. Non-Goals

This project does not attempt to replace:

- Qifa
- Docker
- Kamal Proxy
- Let's Encrypt / lego
- Cloudflare
- SOPS

It is not a Kubernetes distribution, infrastructure orchestrator, or hosted deployment platform.

It provides **conventions, documentation, and reusable agent knowledge** for operating existing tools effectively.

## 11. Future Evolution

Potential additions, only when justified:

- Server bootstrap automation.
- Configuration validation scripts.
- Deployment smoke tests.
- Certificate expiration monitoring.
- Git-based configuration reconciliation.
- Additional agent skills.
- Reusable templates for common deployments.

Avoid building a framework before the operational need exists.

---

## Guiding Principle

**One server, one clear deployment configuration boundary, minimal moving parts.**

Let Qifa manage deployments and certificates, Kamal Proxy manage routing, Cloudflare manage public ingress, and SOPS manage encrypted secrets.

Keep infrastructure understandable enough that both humans and AI agents can maintain it safely.