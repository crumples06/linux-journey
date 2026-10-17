# Week-18
*12/10/26-18/10/26*

## CI for CA testing

I started making a CI pipeline for my segmented-bastion-ca project. The main problem is that the CI starts from nothing. A github runner is a blank machine, it has no data from `ca_data` volume, my CA keys, or my vault password. So the pipeline has to build a throwaway copy of the whole lab, test it, and discard it.

First step i added was a simple `docker compose up -d ca --wait` to start up the ca container. Then i added a step to show logs if the previous step failed. 
Now i had to copy the user and host public ca key to the host from the ca container. So i just copied them from `/host/step/certs/ssh_user_ca_key.pub` from ca container to the host at `ca/user_ca.pub` (same for host ca key). 

Next, i want to create a `known_hosts` file in the ca folder (in host). To configure clients to accept host certificates I need to add the host CA public key in `known_hosts` file with the following format:
`@cert-authority *example.com ecdsa-sha2-nistp256 AAAAE...=`
Where `*.example.com` is a pattern that matches the hosts and `ecdsa-sha2-nistp256 AAAAE...=` should be the contents of the host CA public key.
So i ran this command: `echo "@cert-authority [localhost]:2222,bastion,server1,server2 $(cat ca/host_ca.pub)" > ca/known_hosts` in the step.

Next i wanted to capture the CA root fingerprint, it was to be set up as an environment variable, 
```
FP=$(docker compose exec ca -T step certificate fingerprint /home/step/certs/root_ca.crt)
echo "CA_FINGERPRINT=$FP" >> "$GITHUB_ENV"
```

### CA root certificate

It is a self-signed X.509 certificate. It contains the root CA’s public key and is signed by the root CA's own public key. 
Root fingerprint is a unique hash of that document used to verify the CA during bootstrapping.

1. The admin obtains the root fingerprint from a trusted source: `step certificate fingerprint $(step path)/certs/root_ca.crt`
2. Admin gives that fingerprint to the client out-of-band and the CA URL
3. The client bootstraps to the CA using th URL
4. The client connects to the CA URL and downloads the root certificate.
5. The client computes the SHA-256 hash of that downloaded certificate.
6. It compares its computed hash to the fingerprint already provided.
7. If they match, the client trusts the certificate and installs it. If they don’t match, it aborts.


Now the next step involved writing new CI `group_vars` as the file is encrypted and also wont be useful anyways since CI is running on a blank runner from start. I added `ca_fingerprint`, `ca_provisioner_password`, and `host_cert_path` in the new temporary `vars.yml` in `control/group_vars/all/vars.yml`.
Next i built the rest of the containers and waited for their `sshd` to start listening.

The next issue was about `ssh_config_control`. The `ssh_config_control` trusts only the host CA, with strict checking. On a fresh lab, no host has a certificate yet, so they present plain keys and strict checking rejects them. The playbook then fails at its first task, before it can issue the certificates that would fix this.
So i had to make a new temporary `ssh_config_control` file for this runner. In this new config, i have to relax host checking.
```
- name: Write bootstrap ssh config
        run: |
          cat > control/ssh_config_bootstrap <<'EOF'
          Host bastion server1 server2
            User admin
            IdentityFile ~/.ssh/id_ed25519
            CertificateFile ~/.ssh/id_ed25519-cert.pub
            IdentitiesOnly yes
            UserKnownHostsFile /ca/known_hosts
            StrictHostKeyChecking yes

          Host server1 server2
            ProxyJump bastion
          EOF
```

That config turned out to be wrong. I had copied the strict settings into it (`UserKnownHostsFile /ca/known_hosts` and `StrictHostKeyChecking yes`), so it behaved exactly like `ssh_config_control` and would have failed at the same place. The two lines that matter for the bootstrap are these:

```
- name: Write bootstrap ssh config
        run: |
          cat > control/ssh_config_bootstrap <<'EOF'
          Host bastion server1 server2
            User admin
            IdentityFile ~/.ssh/id_ed25519
            CertificateFile ~/.ssh/id_ed25519-cert.pub
            IdentitiesOnly yes
            UserKnownHostsFile /tmp/bootstrap_known_hosts
            StrictHostKeyChecking accept-new

          Host server1 server2
            ProxyJump bastion
          EOF
```

I could have passed these two settings as `-o` options on the command line, but they would not have worked. `-o` options don’t reach the hidden first-hop connection that ProxyJump creates, the same problem I hit earlier with `-i`. A whole config file passed with -F is read by both hops, so the relaxed settings have to live inside the file.

### The bootstrap trust gap

On a fresh lab the control node is configured to trust a host only if it presents a certificate signed by the host CA. But no host has a certificate yet, because the playbook is what issues them. So the playbook can’t connect to the hosts to create the thing that would let it connect. This only happens on the very first run, and again whenever the host key volumes are wiped.

It’s also a real trust question and not just an inconvenience. On the first connection there is no certificate and no stored fingerprint, so the host’s key has to be accepted on faith. That is the trust-on-first-use behavior the project replaced earlier. In CI this is harmless, because everything happens inside one Docker network on a runner that is destroyed afterwards. So the relaxed config is used for exactly one playbook run, and everything after it is strict.

The next step runs the playbook with the bootstrap config. The inventory already sets `ansible_ssh_common_args`, and extra vars have the highest precedence in Ansible, so passing it with `-e` overrides it for this run only:

```
- name: Run Ansible Playbook (bootstrap)
  run: |
    docker compose exec -T control ansible-playbook -i inventory.yml site.yml \
      -e '{"ansible_ssh_common_args": "-F /work/ssh_config_bootstrap"}'
```

After that I run the playbook a second time with the normal inventory, so it uses the strict `ssh_config_control`:

```
- name: Run ansible playbook again (strict)
  run: docker compose exec -T control ansible-playbook -i inventory.yml site.yml
```

This second run only succeeds if the hosts now present CA-signed certificates that the control node accepts. So it proves the gap was closed by the first run.

### Test user certificate

The tests run from the runner itself, using the repo’s `ssh_config` through the published port, the same way I use the lab. The runner needs an identity, so I generate a fresh key and have the CA sign it from a scratch directory inside the container (to avoid the root-owned file permission problem from the README):

```
- name: Issue test user certificate
  run: |
    mkdir -p keys
    ssh-keygen -t ed25519 -N "" -f keys/key
    printf '%s' "${CI_CA_PASSWORD}" | docker compose exec -T ca sh -c 'mkdir -p /tmp/t && cat > /tmp/t/pw && chmod 600 /tmp/t/pw'
    docker compose cp keys/key.pub ca:/tmp/t/key.pub
    docker compose exec -T ca step ssh certificate --sign --force --provisioner admin \
      --provisioner-password-file /tmp/t/pw --principal admin --not-after 1h ci-user /tmp/t/key.pub
    docker compose cp ca:/tmp/t/key-cert.pub keys/key-cert.pub
```

The key ID `ci-user` is what makes the server logs useful later, because it lets the tests tell this login apart from `ansible-control`.

### Positive tests

A positive test isn’t just “the login worked”. A login can work without the certificate being involved, for example when a static key is still accepted. So each host gets two kinds of check, one for the outcome and one for the mechanism. `ci/positive-tests.sh` does four things:

1. `ssh <host> hostname` is compared with `hostname` run inside that container, so I know I reached the right machine.
2. Each host’s log must contain `Accepted publickey for admin ... ED25519-CERT ... ID ci-user`. Plain `ED25519` would mean a static key got in.
3. `ssh -v` must say `known and matches the ED25519-CERT host certificate`, so the client verified host identity through the CA.
4. `/home/admin/.ssh` must not exist on any host.

### Negative tests

A negative test should pass when the login is refused. But “the command failed” is not enough evidence, because a typo, a timeout or a missing file also makes ssh fail, and the test would stay green for the wrong reason. So the helper requires both a non-zero exit and `Permission denied` in the output:

```
expect_denied() {
  out=$(ssh -F "$cfg" -o BatchMode=yes ... server1 true 2>&1) || rc=$?
  [ "$rc" -ne 0 ] || fail "login unexpectedly SUCCEEDED"
  echo "$out" | grep -q "Permission denied" || fail "failed, but not with Permission denied"
}
```

Each case also gets its own complete ssh config, generated by a small function, so both `ProxyJump` hops use the same bad identity. The cases are:

- a valid key with no certificate (a fresh key, because OpenSSH would auto-load a `-cert.pub` sitting next to my test key)
- a certificate from a rogue CA I generate on the runner, with principal `admin`
- a certificate from the real CA with principal `notadmin`
- password authentication (public-key auth turned off on the client)
- an expired certificate
- a direct connection to server1’s container IP, which must time out

The expired certificate is issued at the top of the script, so it can age while the other tests run.

### Problems I hit in CI

The first problem was the expired certificate itself. Signing failed with `requested duration of 2m0s is less than minimum accepted duration for selected provisioner of 5m0s`. The `admin` provisioner refuses SSH certificates shorter than five minutes. The request also showed up as two minutes, not one, probably because `step` backdates the start time. I didn’t want five extra minutes in every CI run, so I lowered the minimum on the throwaway CI CA only: the workflow edits the `admin` provisioner’s claims in `ca.json` and restarts the CA. The restart doesn’t change any keys, so the exported public keys and the fingerprint stay valid. The lab CA is untouched.

The second problem was the password. The CA was initialized with `ci-throwaway`, but I had defined a job-level `CI_CA_PASSWORD` with a different value, so the negative tests signed with a password the CA didn’t know. The result was the same `failed to decrypt JWE: invalid password error` from the README. The password was hardcoded in three different places, and I had only changed one. The fix was a single definition, `CI_CA_PASSWORD`, read by the compose override `(${CI_CA_PASSWORD:?...})`, the generated `group_vars`, and every signing step. I also stopped using `$` in the value, because `$$` means a literal `$` in compose and the shell treats `$` as special too.

The third problem was a typo. One step used `${CI_CAPASSWORD}`, which doesn’t exist, so it expanded to nothing and wrote an empty password file. The error looked unrelated: `error allocating terminal: open /dev/tty`. `step` found no password and tried to prompt a human on a runner that has no terminal. An empty password causes that, while a wrong password gives the JWE error. A guard like : `"${CI_CA_PASSWORD:?...}"` turns this kind of typo into a readable message.

### Checking that a test can fail

A green run only means something if the tests can go red. On a throwaway branch I commented out the `sleep` before the expired-certificate test and re-ran. The earlier checks passed and then:
```
FAIL: expired certificate: login unexpectedly SUCCEEDED
```

This showed two things. The check really does notice when a certificate is not being rejected. And the certificate was valid when it was issued, because the login succeeded with it. So when the sleep is back, the only thing that has changed is time. I restored the sleep and deleted the branch.

### Finishing the workflow

The last two steps run at the end of the job. On failure, all containers’ logs are dumped, because the sshd `Accepted` and rejection lines are how a failure gets diagnosed. Then the lab is torn down whether or not anything failed:

```
- name: Dump logs on failure
  if: failure()
  run: docker compose logs

- name: Tear down
  if: always()
  run: docker compose down -v
```

The order matters: `down -v` destroys the evidence, so the logs come first. I also added `workflow_dispatch` next to the pull-request trigger, so I can run the pipeline by hand while debugging, without pushing a commit to a branch with an open PR.

The full pipeline now builds a fresh CA, issues host certificates through Ansible, and runs the positive and negative tests from a blank runner. All of it passes, and it tears itself down at the end.

