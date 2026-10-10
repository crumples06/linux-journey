# Ansible Vault & Secrets

Handling secrets in an Ansible project, plus the variable-loading traps that came with it. Project context: `projects/ansible-devops-infra.md`.

## Workflow

```bash
# 1. Password file, added to .gitignore BEFORE it is created
echo "<generated password>" > control/vault_pass.txt

# 2. Put the secret in its own file and encrypt the whole file
ansible-vault encrypt roles/db/vars/vault.yml --vault-password-file vault_pass.txt

# 3. Check decryption independently of any playbook run
PAGER=cat ansible-vault view roles/db/vars/vault.yml --vault-password-file vault_pass.txt

# 4. Run with the vault password
ansible-playbook -i inventory.ini playbook.yaml --vault-password-file vault_pass.txt
ansible-playbook -i inventory.yml site.yml --ask-vault-pass      # prompt instead of a file
```
`ansible-vault create <file>` makes a new encrypted file directly.

## The `vault_` prefix convention

Keep the encrypted value under a `vault_` name and expose it through a plain name in an unencrypted vars file:
```yaml
# vars/vault.yml  (encrypted)
vault_db_password: "..."

# vars/main.yml   (plaintext)
db_password: "{{ vault_db_password }}"
```
Tasks only ever use `db_password`. Anyone grepping the repo can see where the real value lives without being able to read it.

## Variable loading traps (these cost real time)

- **Ansible only auto-loads `vars/main.yml` in a role.** Any other file in `vars/` (like `vault.yml`) is inert until loaded explicitly: `vars_files:` at play level, or `include_vars` at task level.
- The file still decrypts fine with `ansible-vault view`, so the check in step 3 passes while the variable is undefined at runtime. The failure only appears when a task actually references it. A secret nobody uses yet can be broken from day one without anyone noticing.
- **Role order matters for variable availability.** A variable defined by role B is not visible to role A if A runs first in the same play.
- A role's secrets should live with that role (`roles/db/vars/vault.yml`), not in an unrelated role.

```yaml
- hosts: db
  become: yes
  vars_files:
    - roles/db/vars/vault.yml
  roles:
    - db
    - hardening
```

## Vault only protects data at rest

At runtime the secret is in memory. Any task that handles it should carry `no_log: true`, otherwise verbose output can print it. Passing it via `stdin:` to a command that reads `/dev/stdin` keeps it off disk too.

## Vault in CI (GitHub Actions)

- `--vault-password-file` takes a **file path**, never the secret itself.
- Store the password as a repository secret and write it to a file on the runner before the playbook step.
- Runners are ephemeral VMs destroyed after the job, so nothing written there needs manual cleanup.
- See `ci-cd/github-actions.md`.

## Four different passwords (CA automation)

These are easy to mix up. The question to ask is "who is authenticating to what".

| Secret | What it protects | Where it lives |
|---|---|---|
| Vault password | The `vault.yml` file | Supplied at run time (`--ask-vault-pass` or a gitignored file) |
| CA provisioner password | The CA's willingness to sign certificates | Inside `vault.yml`, passed to `step` by the task |
| CA key password | The CA's own private keys | A file inside the CA container, read at startup |
| SSH key passphrase | A private key file on disk | Disabled for the automation key (`--no-password --insecure`) |

Mixing up the provisioner password and the CA key password gives `failed to decrypt JWE: invalid password`.

## Leaked or committed secrets

See `git.md`. Short version: treat a leaked credential as compromised and rotate it.
