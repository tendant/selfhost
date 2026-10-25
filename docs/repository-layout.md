# Repository Layout

Production configuration lives in a **private** infrastructure repository,
separate from application source code and from this standards repository.

## Server-first

One top-level directory per server. When a server runs one deployment unit,
its configuration sits directly in the server directory:

```text
infra/
├── AGENTS.md
├── .sops.yaml
├── .gitignore
│
├── prod-01/
│   ├── qifa.yaml
│   ├── secrets.sops.env     # app secrets
│   └── cloudflare.sops.env  # DNS token for certificate issuance
│
└── dev-01/
    ├── qifa.yaml
    ├── secrets.sops.env
    └── cloudflare.sops.env
```

When a server runs several independently deployed units, give each its own
subdirectory. Server-scoped files stay at the server level:

```text
prod-01/
├── cloudflare.sops.env
├── project-a/
│   ├── qifa.yaml
│   └── secrets.sops.env
└── project-b/
    ├── qifa.yaml
    └── secrets.sops.env
```

## Deployment units

A deployment unit is anything deployed, rolled back, and versioned on its own,
which in practice means one `qifa.yaml`.

- Do **not** split API, web, and worker into separate directories if they are
  always deployed together. Use Qifa roles (`servers.<role>`) in one config.
- Do split them if they ship on independent schedules.
- Databases and caches that belong to one app are Qifa `accessories` in that
  app's config.

## No premature sharing

Duplicate a few lines of `qifa.yaml` across servers rather than inventing a
templating layer. Introduce shared files only when real duplication makes
maintenance painful, and document the reason when you do.

## Required files

**`.sops.yaml`** routes encryption keys by path. See [secrets.md](secrets.md).

```yaml
creation_rules:
  - path_regex: (^|/)prod-[^/]+/.*\.sops\.env$
    age: age1prod...,age1ops...
  - path_regex: (^|/)dev-[^/]+/.*\.sops\.env$
    age: age1dev...,age1ops...
```

**`.gitignore`** blocks plaintext secrets:

```gitignore
.env
*.env
!*.sops.env
*.dec
.qifa/
```

**`AGENTS.md`**: copy [templates/AGENTS.md](../templates/AGENTS.md).

## Naming

- Server directories: short, stable, role-based names (`prod-01`, `dev-01`),
  matching the SSH host alias where possible.
- Unit directories: the Qifa `service` name.
- Use the configuration filenames the installed Qifa version expects
  (`qifa.yaml` today).
