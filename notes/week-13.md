## Week-13
*7/9 - 13/9*

## docker in termux
I made the gitea docker compose file and tested it on my laptop, but when i went to put it in termux i encountered a problem, i can't install docker on termux. 

Docker needs kernel features (cgroups, namespaces) that Android's kernel is usually compiled without. I looked for any way around this and i came across a github repository that explains how to do it. It involved a lot of big and complicated commands. 
Th general idea is to use QEMU (a CPU/hardware emulator) to simulate an entire x86_64 computer, install a lightweight Linux distro (Alpine) onto that virtual machine, and run Docker inside it like a normal Linux box.


**Ansible/DevOps Project — Day Log**

**What I worked on:** Got a two-node web tier fully provisioned and verified via Ansible, and mapped out where the load balancer goes next.

**What I built:**
- Fixed up the `web` role's `tasks/main.yml` — install nginx → deploy `index.html` (from `roles/web/files/`) → start + enable the nginx service. Caught a couple of bugs along the way: a wrong `service` module parameter (`start` instead of `state`), and reordered tasks so the webpage lands before nginx starts.
- Learned the difference between a *playbook* (orchestration — maps hosts to roles) and a role's `tasks/main.yml` (the actual task list that gets spliced into the play at runtime).
- Ran `ansible-playbook -i inventory.ini playbook.yaml` against `web1` and `web2` — clean run, `changed=3` on both hosts, no failures.
- Verified it actually worked by curling into the containers directly (`docker exec web1 curl localhost` and via the control container over the shared Docker network) and confirmed my own HTML was being served, not nginx's default page.

**What I understood better:**
- Traced the full SSH/auth chain end to end: `useradd` creates the `ansible` user → `chpasswd` sets its login password → sshd (running via `CMD` in the target Dockerfile) accepts it → `sshpass` + `ansible_password` in the inventory lets Ansible authenticate non-interactively → `NOPASSWD:ALL` in sudoers lets privilege escalation happen without a second prompt.
- Clarified that two identical web nodes running the *same* role isn't redundant busywork — it's what makes load balancing and high availability demonstrable later (kill one, traffic still flows through the other).
- Worked out where application files belong in an Ansible project (inside the role, under `files/` or `templates/`) versus where that breaks down for large real-world apps (git-clone-on-target, pre-built artifacts, or containerized apps instead of raw file copying) — noted as a "figure out later" problem, intentionally deferred for now.

**Known rough edges (intentional, for now):**
- Password-based SSH auth with username-as-password, and passwordless sudo — both flagged as the "before" state I'll harden later as part of the security layer, not something to fix yet.

**Next up:** Building a load balancer node — new nginx role using `upstream`/`proxy_pass` to distribute requests across `web1`/`web2`, deciding whether it needs its own container or its own inventory group.
