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
age-keygen -y ~/.config/sops/age/keys.txt   # public key: a recipient
```

Prefer self-contained files. SOPS stores each file's recipients inside it
(`sops_age__list_N__map_recipient` lines), so decrypting, editing, and adding or
removing recipients work from the file alone. Recipients are chosen once, when
a file is created, per server or environment: prod files list `age1alice...`
and `age1ci...`; dev files add `age1bob...`.

A `.sops.yaml` with `creation_rules` is optional. It only saves passing
recipients when creating files; it is not needed to read or edit them.

## Daily use

```bash
sops edit prod-01/secrets.sops.env          # edit an existing file

# create a file: copy the recipients of an existing file for that server
R=$(grep '^sops_age__list_[0-9]*__map_recipient=' prod-01/cloudflare.sops.env | cut -d= -f2 | paste -sd, -)
sops edit --age "$R" prod-01/myapp/secrets.sops.env

# import an existing plaintext file
sops encrypt --age "$R" --input-type dotenv --output-type dotenv plain.env \
  > prod-01/myapp/secrets.sops.env && rm plain.env
```

Name encrypted files `*.sops.env`. SOPS picks the dotenv format from the `.env`
extension; a name like `.env.sops` is parsed as JSON and fails to decrypt
without extra `--input-type`/`--output-type` flags. With a `.sops.yaml`, drop
`--age` and pass `--filename-override <destination>` when importing, so the
path rules match the destination rather than `plain.env`.

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

1. `sops rotate -i --rm-age <their key> <file>` for every affected file (also
   rotates the file's data key) and commit. Add a key the same way with
   `--add-age`. With a `.sops.yaml`, update it too, or use `sops updatekeys`.
2. Rotate **every secret value** they could decrypt. Removing a key does not
   revoke access to old Git history.

## Recovery

- Keep at least two recipients on every path, including one offline break-glass
  age key stored outside the infrastructure (password manager or printed and
  stored securely).
- Losing every private key for a path means the secrets are unrecoverable;
  regenerate them at their providers.
- Test the break-glass key periodically: `SOPS_AGE_KEY_FILE=<backup> sops decrypt <file> > /dev/null`.
