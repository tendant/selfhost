# selfhost

**Opinionated, agent-friendly infrastructure practices for self-hosting applications.**

Conventions, documentation, and a reusable AI agent skill for running Docker
apps on independent servers with:

| Concern | Tool |
|---|---|
| Deployment, rollback, certificates | [Qifa](https://github.com/tendant/qifa) |
| Ingress and traffic switching | [Kamal Proxy](https://github.com/basecamp/kamal-proxy) |
| DNS, edge TLS, WAF | Cloudflare |
| Certificates | Let's Encrypt (DNS-01 via Cloudflare) |
| Secrets | SOPS + age |

No Kubernetes, no central controller, no shared certificate storage.

## Contents

- [docs/architecture.md](docs/architecture.md): design principles and overall architecture
- [docs/repository-layout.md](docs/repository-layout.md): how to organize a private infrastructure repository
- [docs/deployment.md](docs/deployment.md): deployment standards and workflow
- [docs/tls.md](docs/tls.md): Cloudflare and Let's Encrypt DNS-01 setup
- [docs/secrets.md](docs/secrets.md): SOPS + age secrets management
- [skills/selfhost/](skills/selfhost/SKILL.md): agent skill for designing, reviewing, and changing self-hosted infrastructure
- [templates/AGENTS.md](templates/AGENTS.md): starter `AGENTS.md` for consuming infrastructure repositories

## Using it

1. Create a **private** infrastructure repository following
   [docs/repository-layout.md](docs/repository-layout.md).
2. Copy [templates/AGENTS.md](templates/AGENTS.md) into it and pin the
   standards version.
3. Install the skill for your agent, e.g. for Claude Code:

   ```bash
   cp -r skills/selfhost ~/.claude/skills/
   ```

This repository holds standards only. Never put credentials, server
inventories, or private hostnames here.

## License

See [LICENSE](LICENSE).
