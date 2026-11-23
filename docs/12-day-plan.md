# Cloud-1: 12-day study and build plan

Written Mon 2026-11-23. Replaces the pacing in [`learning-pipeline-design.md`](learning-pipeline-design.md); requirements come from [`subject-checklist.md`](subject-checklist.md) and the subject PDF.

## Assumptions (correct these first)

| Assumption | Value | If wrong |
|---|---|---|
| Evaluation date | On or right after Sat 2026-12-05 | Earlier eval: apply the cut list now |
| Working days | Mon–Sat, Sunday 29 Nov off | Working Sunday gives one more buffer session |
| Session length | 2 h | Shorter: cut list items 1–5 apply from Day 1 |
| Sessions | 11 working (Days 1–11) + Day 12 as buffer or eval day | Total 22 h planned, 24 h maximum |
| Cloud account | You can log into the OVH account as root/owner with your own email | **Blocker, check on Day 1.** The subject stops the defense otherwise |
| Target | OVH Public Cloud, Ubuntu 22.04, hourly billing, deleted after every session | |
| Repo | Public, branch `main` | |

## How a session runs

| Phase | You | Claude |
|---|---|---|
| Start (5 min) | Create a fresh `cloud1-` instance, put its IP in the git-ignored inventory | Asks yesterday's 3 defense questions while it boots |
| Learn | Read the linked doc section | Explains, answers, draws the packet or task flow |
| Build | Write every line of the milestone | Reviews your diff, points at bugs, never writes milestone code |
| End (10 min) | Run the playbook twice, **delete the instance**, add one line to the AI-usage notes in `README.md`, commit yourself | Read-only check that no `cloud1-` instance is left |

Rules that hold every day:

- Every resource is named `cloud1-*`. Touch nothing else in the shared project.
- No secret, openrc file, project ID or real IP in git. Use `203.0.113.x` in examples.
- From Day 2 on, the playbook is run twice per session. A second run that is not `changed=0` is fixed the same day.
- Compose and nginx work is done on local Docker first (free, fast), then deployed.

## Decisions fixed now to save time

| Topic | Decision | Why |
|---|---|---|
| Roles | `common` (base + ufw), `docker`, `app` | Three roles are enough to defend "relevant roles" |
| Compose file | A plain `docker-compose.yml` in the repo, values through `${VAR}`; only `.env` is templated from vault | The subject asks for a file with that exact name; a plain file is easier to explain |
| phpMyAdmin routing | Path: `/` is WordPress, `/phpmyadmin/` is phpMyAdmin | One hostname, so one certificate per host |
| TLS | certbot, HTTP-01, **Let's Encrypt staging by default**, production from Day 10 only | Rebuilding daily would hit the limit of 5 identical certificates per 7 days |
| DNS | Two DuckDNS names (one per host); the playbook updates the record | The IP changes at every rebuild |
| WordPress setup | Installed by the playbook with WP-CLI, no browser wizard | "No manual steps" in the checklist Verify line |

## Front-loaded blockers (all on Day 1)

- [ ] Confirm you can log into OVH as the account owner (see assumptions).
- [ ] DuckDNS account, two subdomains, token noted outside the repo.
- [ ] Dedicated SSH keypair, public key uploaded to OVH as `cloud1-key`.
- [ ] Flavor and hourly price checked in the OVH console; pick the smallest with 2 GB RAM.
- [ ] `uv run ansible --version` works.
- [ ] `.gitignore` in place before the first real inventory exists.

---

### Day 1 — Mon 2026-11-23

- **Goal:** Clear every blocker and reach a fresh instance with Ansible.
- **Learn:**
  - Inventory and connection variables: [Ansible inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
  - Ad-hoc commands and modules: [Ansible ad-hoc](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html)
  - Flavors and hourly billing: [OVH Public Cloud prices](https://www.ovhcloud.com/en/public-cloud/prices/)
  - Listening sockets, addresses, routes (`ss -tlnp`, `ip a`, `ip r`): [ss man page](https://man7.org/linux/man-pages/man8/ss.8.html)
- **Build:** `.gitignore` (inventory, vault password file, `.env`, openrc, keys), `inventory.example`, `ansible.cfg`. Create, SSH into and delete one instance.
- **Done when:**
  - All six blocker boxes are ticked.
  - `uv run ansible all -m ping` answers `pong` from the instance.
  - You can say what listens on a fresh Ubuntu (`ss -tlnp`) and why.
  - `git status` shows no inventory, key or openrc file; the instance is deleted.
- **Checklist items covered:** §2 Ubuntu 22.04 with only SSH and Python; §4 sizing, stop when unused, no keys committed, billing terms.
- **Defense questions:**
  1. Why delete the instance instead of turning it off?
  2. What does Ansible need on the target, and how does it connect?
  3. Why is the real inventory not in the repository?
- **Time split:** 30 min blockers / 25 min learn / 50 min build / 15 min create and delete.

### Day 2 — Tue 2026-11-24

- **Goal:** Write a first playbook and understand idempotency.
- **Learn:**
  - Plays, tasks, modules: [Playbook intro](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
  - `become`: [Privilege escalation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
  - `--check` and `--diff`: [Check mode](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html)
  - `changed_when`, `creates`: [Error handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)
- **Build:** `site.yml` that waits for first boot to finish (cloud-init, apt locks), refreshes the apt cache and installs base packages. One deliberate `command` task, first non-idempotent, then fixed.
- **Done when:**
  - One `uv run ansible-playbook site.yml` works on a fresh instance.
  - The second run ends with `changed=0`.
  - You can explain each task's module and why it reports `ok` or `changed`.
- **Checklist items covered:** §1 every stage automated; §2 idempotent; §2 script assumes only Ubuntu 22.04.
- **Defense questions:**
  1. What does idempotent mean, and how does a module achieve it?
  2. Why prefer `apt` over `shell: apt-get install`?
  3. What can go wrong with apt on a just-booted cloud instance?
- **Time split:** 35 min learn / 70 min build / 15 min create and delete.

### Day 3 — Wed 2026-11-25

- **Goal:** Move to roles and close the host firewall to 22/80/443.
- **Learn:**
  - Role layout, defaults: [Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
  - Handlers: [Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
  - ufw defaults and rules: [Ubuntu firewall guide](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
  - The Ansible module: [community.general.ufw](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html)
  - Theory, 15 min only: a packet for the host goes PREROUTING → INPUT; a packet for a container goes PREROUTING → FORWARD. ufw filters INPUT.
- **Build:** `roles/common`: Day 2 tasks moved in, plus ufw (allow 22 **before** enabling, default deny incoming, allow 80 and 443). Optional: sshd drop-in forcing key-only login, with a handler.
- **Done when:**
  - `sudo ufw status verbose` shows deny incoming and only 22, 80, 443 allowed.
  - `nmap <host>` from your machine shows 22 open, 80 and 443 closed (nothing listens yet), the rest filtered.
  - Second run `changed=0`; SSH still works after the firewall is on.
- **Checklist items covered:** §2 roles; §2 only 22/80/443; §2 public access limited.
- **Defense questions:**
  1. Why must the SSH rule exist before ufw is enabled?
  2. What is the difference between `closed` and `filtered` in nmap?
  3. When does a handler run, and how often?
- **Time split:** 35 min learn / 70 min build / 15 min create and delete.

### Day 4 — Thu 2026-11-26

- **Goal:** Install Docker Engine by role and prove how Docker interacts with ufw.
- **Learn:**
  - Docker's apt repository: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
  - Published ports, NAT, ufw bypass: [Docker packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
  - Bridge networks and container DNS: [Bridge network driver](https://docs.docker.com/engine/network/drivers/bridge/)
- **Build:** `roles/docker`: Docker's GPG key and apt repo, `docker-ce`, CLI, containerd, compose plugin; service enabled at boot. Not Ubuntu's `docker.io`.
- **Lab (30 min, on the instance, by hand, then removed):**
  - Run nginx with `-p 8080:80`. Reach it from outside although ufw denies 8080.
  - Find the DNAT rule: `sudo iptables -t nat -L DOCKER -n`.
  - Run a container without `-p`. Show it is unreachable from outside.
- **Done when:**
  - `docker --version` is 28 or later; `docker compose version` works; `systemctl is-enabled docker` says `enabled`.
  - Second run `changed=0`.
  - You can explain in three sentences why port 8080 was reachable.
- **Checklist items covered:** §2 public access limited; §2 only 22/80/443 (the reasoning); §2 roles.
- **Defense questions:**
  1. Why does a published port ignore ufw?
  2. So what really keeps MariaDB off the internet in this project?
  3. Why docker-ce from Docker's repo and not `docker.io`?
- **Time split:** 25 min learn / 50 min build / 30 min lab / 15 min create and delete.

### Day 5 — Fri 2026-11-27

- **Goal:** Create the vault and get MariaDB, WordPress (php-fpm) and nginx working locally over HTTP.
- **No cloud instance today.** All work is on local Docker.
- **Learn:**
  - Encrypting variables: [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
  - Services, `restart`, `depends_on`, healthchecks: [Compose services reference](https://docs.docker.com/reference/compose-file/services/)
  - Named volumes: [Docker volumes](https://docs.docker.com/engine/storage/volumes/)
  - `.env` interpolation: [Compose environment variables](https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/)
  - nginx to php-fpm: [ngx_http_fastcgi_module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
- **Build:**
  - `group_vars/all/vault.yml` (encrypted) with DB and WordPress credentials, created **before** the DB service is written. Plain `vars.yml` maps `vault_*` names.
  - `docker-compose.yml`: `mariadb`, `wordpress` (fpm image), `nginx`. Only nginx has `ports:`. Named volumes for DB data and WordPress files. Restart policy on every service.
  - nginx config serving WordPress through FastCGI.
  - Local test only: a git-ignored `.env` with throwaway values, different from the vault values.
- **Done when:**
  - `docker compose ps` shows three services, one process each; only nginx publishes a port.
  - `curl -I http://localhost` returns a WordPress response (redirect to the installer).
  - `head -1 group_vars/all/vault.yml` shows `$ANSIBLE_VAULT`; `git grep -iE 'password|secret|token'` shows only variable names.
- **Checklist items covered:** §1 one container per process; §2 separate containers that communicate; §2 `docker-compose.yml`; §2 no hard-coded secrets; §2 official images.
- **Defense questions:**
  1. How does WordPress find the database without an IP address?
  2. Why does nginx need the WordPress files volume too?
  3. Where is the vault password, and why is committing the encrypted file acceptable?
- **Time split:** 30 min learn / 85 min build / 5 min notes.
- **Tip:** the FastCGI `SCRIPT_FILENAME` must be the path as seen inside the php-fpm container.

### Day 6 — Sat 2026-11-28

- **Goal:** Deploy the stack to a fresh instance with one playbook run.
- **Learn:**
  - Templates: [Templating with Jinja2](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html)
  - Running Compose from Ansible: [community.docker.docker_compose_v2](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
  - Restart after reboot: [Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)
- **Build:** `roles/app`: project directory on the host, `docker-compose.yml` and nginx config copied, `.env` templated from vault (mode `0600`, `no_log`), stack started with `docker_compose_v2`.
- **Done when:**
  - Fresh instance, one run: `curl -I http://<host>` returns a WordPress response.
  - Second run `changed=0`.
  - From outside: `nc -zv <host> 3306` fails; `nmap <host>` shows 22 and 80 open, 443 closed.
  - `sudo reboot`, wait, the site answers again without action.
- **Checklist items covered:** §2 fully automated; §2 WordPress on an instance; §2 database not reachable; §2 restarts after reboot; §2 roles.
- **Defense questions:**
  1. What happens, step by step, when the instance reboots?
  2. Why is `.env` templated on the host and not committed?
  3. What does `no_log` protect, and from whom?
- **Time split:** 20 min learn / 80 min build / 20 min create, reboot test and delete.

## Week 1 checkpoint (end of Sat 28 Nov)

Must be true:

- [ ] One playbook run on a fresh instance gives WordPress over HTTP.
- [ ] Three roles exist: `common`, `docker`, `app`.
- [ ] Second run ends with `changed=0`.
- [ ] Secrets exist only in the vault; the git grep audit is clean.
- [ ] From outside, only 22 and 80 answer; 3306 does not.
- [ ] The site comes back after a reboot.

| If this is missing | Then |
|---|---|
| One box | Fix it in the first 30 min of Day 7 and apply cuts 1–4 |
| Two or more boxes | Use Day 7 to finish week 1. Apply cuts 1–7, merge Day 7 into Day 8, and use Sunday 29 Nov or Day 12 as a work session |
| No deploy on a fresh instance at all | Apply cuts 1–9 (self-signed TLS). Days 7–9 become: deploy, phpMyAdmin + routing, TLS + two hosts |

### Day 7 — Mon 2026-11-30

- **Goal:** Add phpMyAdmin, route by URL, and install WordPress without the browser wizard.
- **Learn:**
  - How nginx picks a server and a location: [How nginx processes a request](https://nginx.org/en/docs/http/request_processing.html)
  - Reverse proxy: [ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
  - phpMyAdmin image variables (`PMA_HOST`, `PMA_ABSOLUTE_URI`): [phpmyadmin official image](https://hub.docker.com/_/phpmyadmin)
  - Scripted install: [wp core install](https://developer.wordpress.org/cli/commands/core/install/)
- **Build:**
  - `phpmyadmin` service, no published port; nginx `location /phpmyadmin/` proxies to it.
  - A one-shot WP-CLI step run by the `app` role: install the site with title, admin user and URL from variables and vault. It must be skipped when WordPress is already installed.
- **Done when:**
  - `http://<host>/` shows the installed site (no wizard); `http://<host>/phpmyadmin/` shows phpMyAdmin.
  - Logging into phpMyAdmin shows the `wp_*` tables; WordPress admin login works.
  - Second run `changed=0`.
- **Checklist items covered:** §1 several services in parallel; §2 database works with both; §2 URL goes to the right site; §2 fully automated.
- **Defense questions:**
  1. How does nginx decide which container gets a request?
  2. What is the difference between `fastcgi_pass` and `proxy_pass` here?
  3. How does the install step know it has already run?
- **Time split:** 25 min learn / 80 min build / 15 min create and delete.
- **Tip:** the `wordpress:cli` image runs as a different user ID than the php-fpm image. Read the "WP-CLI" part of the [wordpress image page](https://hub.docker.com/_/wordpress) before debugging permissions.

### Day 8 — Tue 2026-12-01

- **Goal:** Serve the site over HTTPS on the DuckDNS name with a staging certificate.
- **Learn:**
  - What a certificate proves, ACME flow: [How Let's Encrypt works](https://letsencrypt.org/how-it-works/)
  - HTTP-01 and why port 80 must be open: [Challenge types](https://letsencrypt.org/docs/challenge-types/)
  - Limits and the staging CA: [Rate limits](https://letsencrypt.org/docs/rate-limits/), [Staging environment](https://letsencrypt.org/docs/staging-environment/)
  - nginx TLS: [Configuring HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html)
  - Record update call: [DuckDNS spec](https://www.duckdns.org/spec.jsp)
- **Build:**
  - Per-host `domain` variable in the inventory; DuckDNS token in the vault.
  - Tasks, in order: update the DuckDNS record to the host's IP; wait until the name resolves to it; obtain the certificate with certbot **before** nginx first starts (guarded so it runs once); start the stack.
  - nginx: 443 with the certificate, 80 redirects to 443. `acme_staging` variable, default true.
  - WordPress URL switched to `https://<domain>`.
- **Done when:**
  - `dig +short <domain>` returns the instance IP.
  - `curl -vk https://<domain>` completes a handshake; the issuer contains `STAGING`.
  - `curl -I http://<domain>` returns a 301 to HTTPS.
  - Second run `changed=0`; `nmap <host>` shows 22, 80, 443.
- **Checklist items covered:** §2 TLS when possible; §2 only 22/80/443; §5 memorable domain over HTTPS; §2 no hard-coded secrets (token).
- **Defense questions:**
  1. How does Let's Encrypt check that you control the domain?
  2. Why use staging during development, and what changes for production?
  3. The certificate lasts 90 days: how would you renew it on a long-lived server?
- **Time split:** 30 min learn / 75 min build / 15 min create and delete.
- **If issuance fails twice:** stop retrying (failed validations are rate-limited too), read the certbot log, and finish the nginx TLS config with a temporary self-signed certificate.

### Day 9 — Wed 2026-12-02

- **Goal:** Prove persistence, portability and parallel deployment on two hosts.
- **Learn:**
  - Forks and the linear strategy: [Controlling playbook execution](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html)
  - Where variables belong: [Using variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)
  - Host keys on rebuilt machines: [Connection details](https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html)
- **Build:**
  - Two instances, `cloud1-a` and `cloud1-b`, each with its own DuckDNS name. One run deploys both.
  - Remove anything host-specific from roles (IP, user, domain). Only the inventory differs.
  - Fix every remaining non-idempotent task. Finish TLS leftovers from Day 8.
- **Done when:**
  - One run, play recap lists both hosts with `failed=0`; second run `changed=0` on both.
  - On one host: create a post with an image and a new user, `sudo reboot`, all three are still there.
  - `git grep -nE '([0-9]{1,3}\.){3}[0-9]{1,3}'` finds no real IP in the repo.
- **Checklist items covered:** §2 several servers in parallel; §2 portable; §2 data survives reboot; §2 restarts after reboot; §2 idempotent.
- **Defense questions:**
  1. Does Ansible deploy the two hosts one after the other or together? How do you know?
  2. Where do uploaded images and user accounts live, and why do they survive?
  3. What would you change to deploy a third server?
- **Time split:** 20 min learn / 80 min build and test / 20 min create and delete two instances.

### Day 10 — Thu 2026-12-03

- **Goal:** Write the README and run the first full rehearsal with production certificates.
- **Learn:**
  - Re-read the subject PDF, chapters II, IV, V and VI (15 min).
  - Port states for the scan report: [nmap port scanning basics](https://nmap.org/book/man-port-scanning-basics.html)
- **Build:**
  - `README.md`: what the project does, how to run it (inventory, vault password, one command), architecture (containers, networks, volumes, ports), design decisions, cost notes, **AI usage** section (what Claude did: tutoring, review, this plan; which files or lines are AI-generated, if any).
  - Check `inventory.example`, file and folder names, `.gitignore`.
  - Rehearsal 1: two fresh instances, `acme_staging` false, the Day 11 table once, untimed. Write down everything that fails.
- **Done when:**
  - A reader can deploy from the README alone.
  - `curl -v https://<domain>` (no `-k`) succeeds on both hosts.
  - You have a written list of rehearsal failures, each fixed or scheduled for Day 11.
- **Checklist items covered:** §3 AI use documented; §5 work in the repo, names checked; §4 all items; first pass over every §2 item.
- **Defense questions:**
  1. Which parts of this repository did an AI produce, and how did you check them?
  2. What does this deployment cost per hour, and how do you avoid surprise bills?
  3. If the evaluator gives you a new empty server, what exactly do you do?
- **Time split:** 15 min learn / 45 min README / 45 min rehearsal / 15 min delete.
- **Certificate budget:** from today, at most one production issuance per domain per day.

### Day 11 — Fri 2026-12-04

- **Goal:** Pass a timed defense rehearsal against every checklist item, from zero.
- **Learn:** nothing new. Claude plays the evaluator.
- **Build:** only fixes found by the rehearsal.
- **Rehearsal script** (start the full port scan early, it takes several minutes):

| # | Step | Proves (checklist) |
|---|---|---|
| 1 | Log into the OVH console as account owner with your own email | §5 root/owner login |
| 2 | Create `cloud1-a` and `cloud1-b`, Ubuntu 22.04, nothing done by hand | §2 fresh instance, Ubuntu-only assumption |
| 3 | Fresh `git clone` of the public repo, add inventory and vault password | §5 work is in the repo; §2 portable |
| 4 | One `uv run ansible-playbook` run on both hosts | §2 fully automated, parallel |
| 5 | Run it again: `changed=0` on both | §2 idempotent |
| 6 | `nmap -p- <host>` from outside: only 22, 80, 443 | §2 ports |
| 7 | `nc -zv <host> 3306` fails | §2 database not reachable |
| 8 | `curl -v https://<domain>`; `curl -I http://<domain>` redirects | §2 TLS; §5 domain |
| 9 | `/` is WordPress, `/phpmyadmin/` is phpMyAdmin; `wp_*` tables visible; admin login works | §2 routing, database with both |
| 10 | `docker compose ps`: four services, one process each | §1, §2 containers |
| 11 | Post with image + new user, `sudo reboot`, everything is back | §2 restart and persistence |
| 12 | `git grep -iE 'password|secret|token'`; show the encrypted vault | §2 no hard-coded secrets; §4 no keys committed |
| 13 | Show roles and README AI-usage section; explain any file Claude picks at random | §2 roles; §3 AI |
| 14 | Delete both instances; console shows no `cloud1-` resource | §4 stop unused services |

- **Done when:** all 14 steps pass in under 60 minutes, and you answered every question without notes.
- **Checklist items covered:** all sections, every item.
- **Defense questions:**
  1. Draw the path of an HTTPS request from the browser to the database.
  2. Which single line, if removed, would expose the database to the internet?
  3. What is not production-ready here, and what would you do next?
- **Time split:** 70 min rehearsal / 35 min fixes and questions / 15 min delete.

### Day 12 — Sat 2026-12-05

- **Goal:** Buffer session, or evaluation day.
- **Learn:** re-read your own README and the defense questions of Days 1–11.
- **Build:** nothing new. If Day 11 failed any step: fix it and rerun only the failed steps plus steps 4–6.
- **If the evaluation is today:**
  - Create the instance(s) 45 minutes before, run the playbook with production certificates, then change nothing.
  - Keep the OVH console login ready in a browser (step 1 of the rehearsal).
  - After the defense: delete every `cloud1-` instance and confirm the list is empty.
- **Done when:** every Day 11 step has passed at least once on a fresh instance, and no instance is running.
- **Checklist items covered:** any item still open; §4 stop unused services; §5 root/owner login.
- **Defense questions:**
  1. Why Ansible and Compose rather than a shell script or Kubernetes?
  2. What happens if you run the playbook while the site is live?
  3. What did you decide yourself, against the AI's first suggestion?
- **Time split:** 60 min fixes or pre-eval deploy / 45 min questions / 15 min delete.

---

## Cut list (apply in this order when behind)

| # | Cut | Cost of cutting |
|---|---|---|
| 1 | `ansible-lint`, tags, image optimisation, per-module notes in `docs/learning/` | None for the checklist. Already unscheduled |
| 2 | netfilter theory beyond "DNAT happens in PREROUTING, before ufw's INPUT" | Weaker answer on one defense question |
| 3 | sshd hardening drop-in (Day 3) | None: the cloud image is already key-only. Check with `sudo sshd -T \| grep passwordauthentication` |
| 4 | Day 1 socket and route mapping, shortened to 10 min | Less networking practice |
| 5 | Separate `internal` backend network; keep one Compose network | None: only nginx publishes ports |
| 6 | Healthchecks and `depends_on` conditions; rely on restart policies | Slower, noisier first start |
| 7 | Automated DuckDNS update; update the record by hand on the website and document it in the README | One manual step outside the server. Explain it at the defense |
| 8 | WP-CLI install; leave the browser wizard | **Risky:** an evaluator may count the wizard as a manual step. Last resort |
| 9 | Let's Encrypt; generate a self-signed certificate with Ansible ([community.crypto.x509_certificate](https://docs.ansible.com/ansible/latest/collections/community/crypto/x509_certificate_module.html)) | TLS still passes ("when possible"). Loses the optional trusted-domain point |
| 10 | Day 12 buffer becomes a work session | No safety margin |

**Never cut:** one-run deploy on a fresh Ubuntu 22.04, idempotency, two hosts in parallel, roles, `docker-compose.yml`, one process per container, WordPress + phpMyAdmin + database working together, URL routing, some form of TLS, only 22/80/443, reboot restart and data persistence, no secrets in code, README AI-usage section, owner login at the defense.

## Top 5 risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | TLS or DNS fails late (rate limits, DNS not updated after a rebuild, HTTP-01 blocked) | Staging by default; production at most once per domain per day from Day 10; playbook waits for DNS before certbot; self-signed fallback (cut 9) |
| 2 | Sessions overrun: instance setup, apt locks at first boot, slow debug loops on a remote host | Create the instance first and learn while it boots; wait-for-cloud-init task from Day 2; Compose and nginx debugged locally; Day 12 kept free |
| 3 | Late surprises in idempotency, reboot or two-host runs | Playbook run twice every session from Day 2; reboot test already on Day 6; two hosts on Day 9, not Day 11 |
| 4 | Leak in the public repo, or a forgotten instance in the shared project | `.gitignore` on Day 1; grep audit before each commit; end-of-session delete plus read-only check; only `cloud1-*` resources are ever touched |
| 5 | Cannot explain a line at the defense (AI chapter: fail) | You type all code; three defense questions per day, re-asked the next morning; AI-usage notes updated daily; random-file explanation in the rehearsal |
