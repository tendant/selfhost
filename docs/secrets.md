# Secrets

Secrets are stored in Git, encrypted with [SOPS](https://github.com/getsops/sops)
using [age](https://github.com/FiloSottile/age) keys. The encrypted files are the
source of truth.

## Rules

- Commit only encrypted `*.sops.env` files. Never commit plaintext `.env`
  files; block them in `.gitignore` (see [repository-layout.md](repository-layout.md)).
- Decrypt only at the moment of use, in a pipe or `sops exec-env`. Never write
  plaintext to disk in the repository.
- Restrict decryption: each environment has its own recipients. Developers who
  only touch `dev-*` cannot decrypt `prod-*`.
- Never print secrets in logs, CI output, or agent transcripts. Don't
  `cat` decrypted files; inspect keys with `sops decrypt ... | cut -d= -f1`.
- Cloudflare API tokens and TLS private keys never go into a public repository.

## Setup

One age key per person and per automation identity (CI runner, ops machine):

```bash
age-keygen -o ~/.config/sops/age/keys.txt   # private key, never shared
age-keygen -y ~/.config/sops/age/keys.txt   # public key: add to .sops.yaml
```

Configure recipients per path in `.sops.yaml` at the repository root:

```yaml
creation_rules:
  - path_regex: (^|/)prod-[^/]+/.*\.sops\.env$
    age: >-
      age1alice...,
      age1ci...
  - path_regex: (^|/)dev-[^/]+/.*\.sops\.env$
    age: >-
      age1alice...,
      age1bob...,
      age1ci...
```

## Daily use

```bash
sops edit prod-01/secrets.sops.env          # create or edit

# import an existing plaintext file
sops encrypt --filename-override prod-01/secrets.sops.env plain.env \
  > prod-01/secrets.sops.env && rm plain.env
```

Name encrypted files `*.sops.env`. SOPS picks the dotenv format from the `.env`
extension; a name like `.env.sops` is parsed as JSON and fails to decrypt
without extra `--input-type`/`--output-type` flags. `--filename-override` makes
the `.sops.yaml` path rules match the destination, not `plain.env`.

Qifa decrypts at deploy time; the command runs on the deployer, relative to the
config file:

```yaml
env:
  secret_command: sops --decrypt secrets.sops.env
```

Output must be `KEY=VALUE` lines. Values from `secret_command` override
`env.clear` on collision. `qifa env diff` shows whether running containers have
the current values.

For non-Qifa commands that need secrets (certificate issuance), use
`sops exec-env <file> '<command>'` so values exist only in that process's
environment.

## CI

Give the CI runner its own age key, stored as a CI secret
(`SOPS_AGE_KEY`), and add its public key only to the paths it deploys.

## Rotation

**A secret value** (password, token):

1. Create the new credential at the provider.
2. `sops edit` the file and commit.
3. `qifa deploy` affected units; confirm with `qifa env diff`.
4. Revoke the old credential.

**A recipient** (person leaves, key compromised):

1. Remove their public key from `.sops.yaml`.
2. `sops updatekeys <file>` for every affected file and commit.
3. Rotate **every secret value** they could decrypt. Removing a key does not
   revoke access to old Git history.

## Recovery

- Keep at least two recipients on every path, including one offline break-glass
  age key stored outside the infrastructure (password manager or printed and
  stored securely).
- Losing every private key for a path means the secrets are unrecoverable;
  regenerate them at their providers.
- Test the break-glass key periodically: `SOPS_AGE_KEY_FILE=<backup> sops decrypt <file> > /dev/null`.
