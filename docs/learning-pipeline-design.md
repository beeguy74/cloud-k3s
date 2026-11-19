# Learning pipeline design

Date: 2026-11-19. Status: agreed in conversation; module contents are detailed per module later.

## Intent

Learn everything the Cloud-1 project needs **while building it**, so every line in the repository can be explained and defended at the evaluation. The subject's AI chapter requires this: a student who cannot explain AI-generated design decisions fails.

The learner writes the project code. Claude acts as tutor and reviewer: explains concepts, points to official docs, sets labs, reviews code, asks defense-style questions. Claude does not write the milestone code.

## Learner profile

| Area | Level | Depth in the pipeline |
|---|---|---|
| Docker | Comfortable | Short refresher, Compose specifics only |
| Linux, SSH | Comfortable | Only SSH keys for Ansible and SSH hardening |
| Networking | Used a lot, theory weak | Deep: from first principles |
| Ansible | New | Deepest: from zero to roles, vault, idempotency |

## Goals, non-goals, anti-goals

**Goals**
- Build a project that passes every item in [`subject-checklist.md`](subject-checklist.md).
- Every module produces a real piece of the repository and ticks checklist items.
- Understand the networking behind the stack well enough to debug it, not just run it.

**Non-goals**
- Linux and SSH basics, Kubernetes, Docker image optimisation.
- Automating instance creation (the subject assumes an existing Ubuntu 22.04 host with SSH and Python).

**Anti-goals** (failure even if the site works)
- Code in the repo that the learner cannot explain.
- A forgotten instance running up costs.
- Secrets or cloud account identifiers committed to the (public) repository.
- Any impact on other resources in the shared cloud project.

## Decisions

- **Stack:** Ansible + Docker Compose. k3s was dropped: the subject requires a `docker-compose.yml`, and k3s exposes extra ports (6443, 10250, 8472/udp, NodePorts) that conflict with "only 22/80/443".
- **Branch:** work happens on `compose`.
- **Targets:** OVH Public Cloud instances (Ubuntu 22.04). No Vagrant. Local Docker is used for quick, free networking and Compose experiments.
- **Controller:** this machine, with Ansible pinned in `pyproject.toml` and run as `uv run ansible-playbook ...`.
- **Who runs what:** the learner runs every command that creates, changes or deletes cloud resources. Claude writes those commands and only runs read-only checks (for example `openstack server list`).
- **Pace:** about 10 modules of 4–6 hours each, self-paced. If a deadline appears, modules get merged or cut to fit.

## Cloud guardrails

The OVH project is shared with other workloads, so:

- Prefix every resource with `cloud1-` (instances, SSH keypair, security group).
- Use a dedicated SSH keypair for this project.
- Use **hourly** billing and the smallest flavor that runs the stack.
- **Delete** the instance at the end of each session. On OVH, *turn off*, *pause* and *suspend* are billed at full price; only *shelve* (snapshot billed) and *delete* reduce cost. Rebuilding from scratch each session also tests the "fresh instance" requirement.
- Keep `openrc.sh`, project IDs and real host IPs out of the repo. The real inventory is git-ignored; the repo holds an `inventory.example`.

## Approaches considered

| Approach | Trade-off |
|---|---|
| **Build-along** (chosen) | Each module: concepts → small lab → real project milestone. Learning is applied immediately and the repo grows from the learner's own work. Slower than letting AI build it. |
| Study-first | Finish Ansible and networking courses, then build. Deep, but knowledge goes stale before use and the project starts late. |
| Just-in-time | Build first, explain on demand. Fastest, but shallow: gaps show up at the defense. |

## Module outline

Each module has: concepts (with official doc links), a lab, a project milestone, the checklist items it covers, and defense-style self-check questions.

| # | Module | Key concepts | Milestone |
|---|---|---|---|
| 0 | Cloud hygiene | OVH console, `openstack` CLI (read-only), flavors, hourly billing, shelve vs delete, keypairs | Create, SSH into and delete a `cloud1-` instance |
| 1 | Networking I | IPs, interfaces, routes, ports, sockets (`ip a`, `ip r`, `ss -tlnp`), TCP handshake, DNS lookup (`dig`) | Map everything listening on a fresh Ubuntu instance |
| 2 | Ansible basics | Inventory, ad-hoc commands, modules vs `shell`, playbooks, facts, variables, `--check`/`--diff`, idempotency | First playbook; second run ends with `changed=0` |
| 3 | Host hardening | netfilter packet flow (PREROUTING → INPUT/FORWARD → OUTPUT), iptables/nftables/ufw, SSH hardening, handlers | `common` role: ufw allows only 22/80/443, key-only SSH |
| 4 | Ansible structure | Roles, defaults vs vars, Jinja2 templates, group_vars/host_vars, tags, `ansible-lint` | `docker` role installing docker-ce (Engine ≥ 28) from Docker's apt repo |
| 5 | Docker networking | Bridge networks, embedded DNS, published vs unpublished ports, NAT and the DOCKER chains, why published ports bypass ufw, `internal: true` networks | Prove with `iptables -t nat -L` and `nmap` which ports are reachable |
| 6 | Secrets | `ansible-vault`, `no_log`, generating `.env` from vault, `.gitignore`, `git grep` audit | DB credentials stored only in vault |
| 7 | Compose stack | MariaDB, WordPress (php-fpm), nginx, phpMyAdmin; 1 process = 1 container; named volumes; healthchecks; `depends_on`; restart policies | `app` role templates `docker-compose.yml` and deploys it with `community.docker.docker_compose_v2` |
| 8 | HTTP routing | HTTP and the Host header, nginx server/location blocks, FastCGI to php-fpm, `proxy_pass`, host vs path routing | WordPress and phpMyAdmin served by URL |
| 9 | TLS and DNS | Certificates, chains, CAs, ACME HTTP-01 vs DNS-01, DuckDNS, renewal, 80 → 443 redirect | Valid HTTPS on a domain |
| 10 | Defense rehearsal | Idempotency, two hosts in parallel, fresh-instance run, reboot test, README with AI-usage section | Every checklist item verified |

## Left open

- Deadline (default pace above until one is given).
- Flavor: picked by the learner from current console prices.
- Domain name (DuckDNS or other).
- Routing scheme (subdomain vs path for phpMyAdmin): decided in module 8.
- Per-module material location: proposed `docs/learning/NN-<topic>.md`, written when each module starts.
