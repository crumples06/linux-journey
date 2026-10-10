# Git: Ignoring, Leaks, and Line Endings

Lessons from a real credential-in-history incident and a Windows line-endings bug.

## `.gitignore` only affects untracked files

- It has **zero effect** on files already committed. Never trust the ignore rule; check:
  ```bash
  git ls-files | grep ssh_keys        # is it tracked?
  git check-ignore -v path/to/file    # which rule (if any) ignores it?
  ```
- Pattern gotcha: `./control/ssh_keys` (leading `./`, no trailing slash) matched nothing. Use `control/ssh_keys/`.
- Add secrets to `.gitignore` **before** creating them (as I did with `vault_pass.txt`).
- Stop tracking without deleting the file:
  ```bash
  git rm --cached control/ssh_keys/ansible_hardening_key
  git rm --cached control/ssh_keys/ansible_hardening_key.pub
  ```
  This only removes it going forward; it stays in history.

## Leaked credentials: rotate, don't scrub

- Treat any leaked key, password or token as **permanently compromised**.
- History rewriting (`filter-repo`, BFG, force-push) is unreliable once anyone has cloned, and gives no guarantee the secret was not already seen.
- Correct remediation: generate a new credential, deploy it, retire the old one. Commit the `.gitignore` fix and the untracking together.
- Auditing tip: when reasoning about where one secret file should live, check whether the same "gitignore protects it" assumption holds for similar files. That is how the committed SSH key was found.

## CRLF line endings corrupting files

- On Windows with `core.autocrlf=true`, Git converts LF to CRLF on checkout for files it thinks are text. For files where exact bytes matter (SSH private keys, certificates, shell scripts) that corrupts them.
- Symptoms seen:
  - `Load key "...": error in libcrypto` before auth is even attempted.
  - `service foo start` -> "No such file or directory": an `execve` failure on a shebang line ending in `\r`.
- Diagnose:
  ```bash
  git config --get core.autocrlf
  cat -A file | head -5            # ^M$ at each line end means CRLF
  ssh-keygen -y -f keyfile         # errors if the key file is corrupt
  ```
- Quick fix on disk: `sed -i 's/\r$//' file`.
- Durable fix, independent of any developer's Git config, via `.gitattributes`:
  ```
  ssh_keys/* -text
  ```
  Broaden it to every place such files live, for example `control/roles/*/files/** -text`, so the class of bug cannot recur project-wide.
