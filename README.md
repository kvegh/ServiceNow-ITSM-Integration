# ServiceNow ITSM Integration

Integrates Red Hat Ansible Automation Platform with ServiceNow across three use
cases, with demo configuration created by automation where its deployment path
has been verified. Licensing and installation prerequisites are documented.

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

- **Repeatable setup.** The initial admin setup account creates a separate
  integration account for everyday jobs. Keep setup access available for
  bootstrap reruns and teardown; deactivation is a separate optional final step
  with a documented way to obtain admin access for later resets. UC1's OAuth
  authorization and automated Flow deployment must be proven, and unavoidable
  manual prerequisites documented.
- **Persistent incident correlation.** Store the original Zabbix problem ID in
  the stock `correlation_id` field. Independent recovery jobs look up the same
  incident, and repeated notifications must not create duplicate tickets.
- **Verified integration permissions.** Test incident read/create/update/resolve
  and the enabled request/CMDB operations using the integration account; the
  scaffold's role list is not a verified permission set.
- **Own content only.** Written from scratch. The
  [shadowman-lab reference repo](https://github.com/shadowman-lab/Ansible-SNOW)
  is consultation material for how ServiceNow APIs behave — do not copy its
  or adapt its files. Its instance-specific custom `u_*` columns and hardcoded callers would
  become inherited debt.
- **Independent of the AIOps self-healing demo.** That demo works today and must
  not be modified. UC2 reuses Zabbix as an event source but brings its own
  rulebook, activation, job templates and trigger action. AAP objects are
  prefixed (see `aap_prefix` in `vars/main.yml`) so the two never collide.

## Layout

| Path | Contents |
|---|---|
| `PLAN.md` | Design plan: phases, use cases, bootstrap breakdown |
| `TODO.md` | Actionable implementation tasks |
| `design_decisions_history.md` | Decisions, rationale, and commit references |
| `docs/` | Prerequisites today; bootstrap and use-case guides as they are built |
| `vars/main.yml` | Non-sensitive config: object names, roles, prefixes |
| `vars/vault.yml.example` | Template for the encrypted secrets file |
| `collections/requirements.yml` | Project-root and local collection requirements |
| `playbooks/`, `roles/`, `extensions/eda/rulebooks/`, `files/updatesets/` | Planned; created as the phases are built |

## Before creating an AAP project for this repo

Use a dedicated `SNOW Demo - Project` that syncs this standalone repository over
HTTPS, with playbook paths relative to this repository root. Replace the
scaffold's umbrella-project default in `vars/main.yml` during implementation,
before bootstrap runs.

AAP resolves project-root `collections/requirements.yml` during sync when
collection downloads are enabled. Earlier notes recorded `community.zabbix` as
missing from both the selected image and the configured collection source.
Verify the current source and resolve this dependency before project sync;
also verify its required Python dependencies in the selected execution image.
Details are in [docs/00-prerequisites.md](docs/00-prerequisites.md).

UC2 is implemented first and runs independently of the Spoke. UC1 setup is gated
on connectivity, TLS trust, OAuth, and automated Flow-deployment verification.

## Local setup and validation

Ansible and an execution environment containing the required collections and
Python dependencies are prerequisites. For local collection setup when needed:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

For documentation changes:

```bash
git diff --check
```

No executable demo, automated test suite, or formatter is configured yet.
Document actual execution and validation commands as implementation lands, and
exercise the bootstrap, reset, and use-case checks in the plan.

## Submodule discipline

This repository is a submodule of
[AAP-advanced-features](https://github.com/kvegh/AAP-advanced-features), which
pins a specific commit. After pushing here, bump the pointer in the umbrella:

    git -C ServiceNow-ITSM-Integration pull
    git add ServiceNow-ITSM-Integration
    git commit -m "bump ServiceNow-ITSM-Integration"
