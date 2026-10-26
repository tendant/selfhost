# Repository Layout

Production configuration lives in a **private** infrastructure repository,
separate from application source code and from this standards repository.

## Server-first

One top-level directory per server. When a server runs one deployment unit,
its configuration sits directly in the server directory:

```text
infra/
├── AGENTS.md
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

## No templates

Every `qifa.yaml` contains literal values: real hostnames, ports, and paths,
read by Qifa exactly as committed.

- No `envsubst`, `${VAR}` placeholders, render steps, or `build/` directories.
- No generator scripts that write configs.
- No Makefile wrappers around Qifa commands. Run `qifa -c <path> <command>`.
- Secrets reach containers only through `env.secret_command`; they are never
  substituted into config files.

Repeated lines across servers or apps are fine. Copying a config and editing
it is simpler to read, review, and debug than any layer that generates it.

## Required files

Each `*.sops.env` file lists its own age recipients, so no `.sops.yaml` is
needed. One is optional, for repositories that want path rules to pick
recipients when creating files. See [secrets.md](secrets.md).

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
