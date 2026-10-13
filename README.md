# ServiceNow ITSM Integration

Integrates Red Hat Ansible Automation Platform with ServiceNow across three use
cases, with every object on every system created by automation.

> **Status: design only. No playbooks have been written yet.**
> This repository currently holds the plan, prerequisites, and variable
> scaffolding. Nothing here has been run. Start at
> [TODO.md](TODO.md) for the next actionable step.

**Design plan: [PLAN.md](PLAN.md)** — phases, directory layout, per-use-case
detail. **Prerequisites: [docs/00-prerequisites.md](docs/00-prerequisites.md)**
— what must exist before anything runs, and the verified collection audit.

## Use cases

| | Flow | Depends on |
|---|---|---|
| **UC1** | Service Catalog item → Ansible Spoke → AAP job → closes the RITM and updates CMDB | IntegrationHub subscription, Spoke installed |
| **UC2** | Zabbix event → EDA → AAP job → creates/resolves a ServiceNow incident | nothing beyond the base setup |
| **UC3** | Compare host reality against CMDB → report and optionally correct discrepancies | nothing beyond the base setup |

UC2 is the intended starting point: it is the only one with no Spoke dependency,
so it reaches a working end-to-end flow fastest.

## Principles

These shaped the design and should not be quietly reversed:

- **Everything automated.** No click-through setup on ServiceNow, AAP or Zabbix.
  The sole exception is the seed service account, because nothing can call the
  API before an API account exists. The bootstrap deprovisions it when done.
- **Own content only.** Written from scratch. The
  [shadowman-lab reference repo](https://github.com/shadowman-lab/Ansible-SNOW)
  is consultation material for how ServiceNow APIs behave — do not copy its
  files. Its instance-specific custom `u_*` columns and hardcoded callers would
  become inherited debt.
- **Independent of the AIOps self-healing demo.** That demo works today and must
  not be modified. UC2 reuses Zabbix as an event source but brings its own
  rulebook, activation, job templates and trigger action. AAP objects are
  prefixed (see `aap_prefix` in `vars/main.yml`) so the two never collide.

## Layout

| Path | Contents |
|---|---|
| `PLAN.md` | Design plan: phases, use cases, bootstrap breakdown |
| `docs/` | Prerequisites today; bootstrap and use-case guides as they are built |
| `vars/main.yml` | Non-sensitive config: object names, roles, prefixes |
| `vars/vault.yml.example` | Template for the encrypted secrets file |
| `collections/requirements.yml` | Local development only — see the warning below |
| `playbooks/`, `roles/`, `extensions/eda/rulebooks/`, `files/updatesets/` | Empty; created as the phases are built |

## Before creating an AAP project for this repo

`collections/requirements.yml` sits on a path AAP treats as magic. At a project
root, AAP runs `ansible-galaxy install` against it during sync, and
`community.zabbix` is neither in `ee-supported-rhel9` nor on the configured
galaxy server — so the sync would fail. Resolve that first; details in
[docs/00-prerequisites.md](docs/00-prerequisites.md). No project exists yet, so
nothing is currently broken.

## Submodule discipline

This repository is a submodule of
[AAP-advanced-features](https://github.com/kvegh/AAP-advanced-features), which
pins a specific commit. After pushing here, bump the pointer in the umbrella:

    git -C ServiceNow-ITSM-Integration pull
    git add ServiceNow-ITSM-Integration
    git commit -m "bump ServiceNow-ITSM-Integration"
