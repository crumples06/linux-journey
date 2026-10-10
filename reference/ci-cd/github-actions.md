# GitHub Actions CI

A pull-request pipeline that builds a Docker stack, runs an Ansible playbook, and verifies behaviour. Project context: `projects/ansible-devops-infra.md`.

## Pipeline shape

1. Trigger on PRs to `main` from a feature branch.
2. Build and start the stack, waiting for healthchecks: `docker compose up --build --wait`.
3. Run the playbook non-interactively inside the control container: `docker exec control ansible-playbook ...`.
4. Verify load balancing.
5. Tear down with `if: always()` so cleanup runs on pass or fail.
6. Branch protection requires the check before merge.

## Healthchecks: `CMD` vs `CMD-SHELL`

- `["CMD", "ansible --version"]` tries to execute a binary literally named `ansible --version`. `CMD` does **not** use a shell.
- `["CMD-SHELL", "..."]` runs through `/bin/sh -c`, so `&&`, pipes and multiple commands work.
- To check several binaries exist, use `command -v`, which is POSIX-portable and explicit:
  ```
  command -v ansible >/dev/null && command -v curl >/dev/null && command -v sshpass >/dev/null
  ```
  `which a b c` has inconsistent multi-argument exit codes across implementations.

## Assert something specific

A single `curl` proving nginx responded does not prove load balancing. Make the page identify its node, request repeatedly, and check both appear:
```yaml
- name: Verify load balancing
  run: |
    for i in $(seq 1 10); do
      docker exec control curl -s http://loadbalancer >> responses.txt
    done
    grep -q web1 responses.txt && grep -q web2 responses.txt
```
(sketch of the pattern; adapt names).
A CI check should assert a particular outcome, not merely "exit 0".

GitHub's default `run` shell uses errexit, so one failing command kills the whole step immediately. For this check that is correct: one failed request should fail the verification.

## Secrets in CI

- Runners are fresh, ephemeral VMs. `actions/checkout` clones the repo, so anything gitignored or untracked (SSH keys, vault password file) is **absent**.
- Rebuild those files from repository secrets before the steps that need them:
  ```yaml
  - name: Write vault password file
    run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > control/vault_pass.txt

  - name: Restore SSH keys
    run: |
      mkdir -p control/ssh_keys
      printf '%s\n' "${{ secrets.ANSIBLE_SSH_PRIVATE_KEY }}" > control/ssh_keys/ansible_hardening_key
      printf '%s\n' "${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}" > control/ssh_keys/ansible_hardening_key.pub
      chmod 600 control/ssh_keys/ansible_hardening_key
  ```
- Worth considering: pass secrets through `env:` and reference `$VAR` in the script, rather than interpolating `${{ secrets.X }}` directly into shell text. It avoids quoting and injection problems.
- No manual cleanup is needed, since the VM is destroyed after the job.

## Fresh containers are never "already bootstrapped"

CI always builds new containers: no SSH key deployed, password auth still on. Long-running local containers have the key trusted from earlier runs, so a playbook that passes locally can fail in CI at *Gathering Facts* with SSH permission denied. Keep a documented password-authenticated bootstrap path for the first run (here, the `ansible_password` line in the inventory).

## `docker exec` in automation

A pseudo-TTY (`-t`) is meaningless in a non-interactive runner; drop `-it` there. Behaviour that is invisible in an interactive terminal matters in CI.

## Failure log

| Symptom | Cause | Fix |
|---|---|---|
| Healthcheck fails immediately | `CMD` with a space-containing string | `CMD-SHELL` |
| Multi-binary check unreliable | `which a b c` | `command -v` chain |
| Gathering Facts: permission denied | Fresh containers, key not yet deployed | Password bootstrap run first |
| "SSH key files absent" | Keys no longer tracked in git, clean checkout lacks them | Restore from secrets |
