# ServiceNow ITSM Integration for AAP

## Context

A standalone ServiceNow ITSM integration demo, built in the `ServiceNow-ITSM-Integration` subdir of the AAP-advanced-features repo, covering three use cases:

1. **UC1 — Service Catalog → Spoke → AAP:** ServiceNow catalog item triggers AAP job via Ansible Spoke, job processes the request and updates/closes the SNow ticket + CMDB
2. **UC2 — Zabbix → EDA → SNow incident:** Zabbix fires an event, EDA triggers an AAP job, Ansible creates/updates/closes a ServiceNow incident
3. **UC3 — CMDB reconciliation:** Playbook that compares infrastructure reality against ServiceNow CMDB and flags/fixes discrepancies

Uses the certified `servicenow.itsm` collection throughout.

### Everything automated

**No click-through setup.** Every object this demo needs — on ServiceNow, on AAP, and on Zabbix — is created by a playbook in this subdir. The demo must be reproducible from scratch against a fresh ServiceNow instance by running the bootstrap in Phase 0.

The single unavoidable exception is the **seed service account**: an admin-scoped ServiceNow user with "Web service access only" enabled, created by hand because there is no API to call before an API account exists. Everything after that is automated, and the bootstrap deprovisions the seed account at the end.

Docs in `docs/` describe how to *run* the automation and what it creates — they are not click-through instructions.

### Own content only

All playbooks, roles, and rulebooks in this demo are written from scratch. The shadowman-lab reference repo (https://github.com/shadowman-lab/Ansible-SNOW) is **consultation material only** — read it when stuck on how a ServiceNow API or module behaves, do not copy or adapt its files. Rationale: its content carries instance-specific assumptions (custom `u_*` columns, hardcoded callers) and a structure built for a different demo; inheriting that imports debt we would have to unpick later.

### Scope boundary — independent of the AIOps demo

This demo is **separate from the AIOps self-healing demo** and must not modify it. The AIOps demo is already working and is duplicated across two locations (`aiops-selfhealing-demo` as its own public repo, and `AAP-advanced-features/AIOps_selfhealing_demo`), so edits there would need syncing in both and risk breaking a demo that currently runs.

UC2 reuses Zabbix as an event source, but with its **own** rulebook, own rulebook activation, and own job templates — all living in this subdir. No AIOps files, workflows, or AAP objects are touched.

**Shared-Zabbix caution:** the AIOps demo already has a Zabbix→EDA path configured. Before wiring UC2, verify what trigger actions / media types / Event Streams already exist so this demo adds its own rather than altering theirs. Prefer a distinct trigger action and a distinct Event Stream, scoped to a host group or tag that the AIOps demo does not use.

## Prerequisites

Only things that cannot be automated — licensing, the seed account, and collection availability. Everything else is Phase 0's job.

### Manual (unavoidable)

- [ ] **Seed service account on ServiceNow** — admin-scoped user, Active=true, Locked out=false, **Web service access only=true**, Password needs reset=false. The "Web service access only" flag is what bypasses SSO/MFA enforcement for machine accounts; without it the API returns 401. This is the only hand-made object; the bootstrap deprovisions it at the end.
- [ ] **IntegrationHub subscription** — licensing gate for the Ansible Spoke (UC1). Cannot be automated; confirm it is active on the instance before planning on UC1.

### Verify before starting

- [ ] **`servicenow.itsm` availability** — not currently synced to the Private Automation Hub's rh-certified repo. Either sync it from console.redhat.com, or declare it in `collections/requirements.yml` and let AAP install it at project sync (the PAH credential already exists).
- [ ] **Other collections** — `ansible.controller` and `ansible.eda` (AAP config-as-code), `ansible.platform` (gateway objects such as the OAuth application), `community.zabbix` (Zabbix trigger action). Confirm each is reachable from the chosen EE.
- [ ] **EE choice** — check whether `ee-supported-rhel9` carries the above. If not, prefer a project-level `collections/requirements.yml` over building a custom EE.
- [ ] **Spoke app installation** — installing the Ansible Spoke from the ServiceNow Store is likely not API-automatable. Verify; if it is a one-time Store install, document it as a prerequisite rather than pretending the bootstrap covers it.

## Phase 0 — Bootstrap (everything as code)

Creates every object the demo needs, on all three systems. Driven by `bootstrap.yml`, which imports the per-system playbooks below so a fresh environment is one command.

### 0.1 Directory structure

```
ServiceNow-ITSM-Integration/
    PLAN.md
    README.md
    bootstrap.yml                         # one entry point: imports all three below
    teardown.yml                          # reverse, for a clean demo reset
    collections/requirements.yml
    vars/
        main.yml                          # non-sensitive config, object names
        vault.yml.example                 # seed creds template
    playbooks/
        bootstrap_servicenow.yml          # 0.2 — SNow-side objects
        bootstrap_aap.yml                 # 0.3 — AAP-side objects
        bootstrap_zabbix.yml              # 0.4 — Zabbix-side objects
        snow_create_incident.yml          # UC2
        snow_update_incident.yml          # UC2
        snow_catalog_fulfill.yml          # UC1
        snow_cmdb_reconcile.yml           # UC3
        snow_cmdb_update.yml              # shared
    roles/
        servicenow_bootstrap/             # SNow account/role/catalog provisioning
        aap_bootstrap/                    # controller + EDA + gateway objects
        zabbix_bootstrap/                 # trigger action, media type
        servicenow_incident/
        servicenow_cmdb/
        servicenow_catalog/
    extensions/eda/rulebooks/
        zabbix-to-snow.yml                # UC2: own rulebook
    files/updatesets/                     # XML update sets for objects with no clean API
    docs/
        00-prerequisites.md               # seed account, licensing, collections
        01-bootstrap.md                   # how to run bootstrap.yml, what it creates
        02-uc1-service-catalog.md
        03-uc2-incident-management.md
        04-uc3-cmdb-reconciliation.md
        05-teardown.md
```

### 0.2 ServiceNow-side bootstrap (`bootstrap_servicenow.yml`)

Runs as the seed account, creates everything else. Uses `servicenow.itsm.api` for generic table CRUD, since the collection's typed modules cover ITSM records but not platform configuration.

- **Service accounts and roles** — the demo's own integration user (separate from the seed), with `rest_api_explorer` plus `catalog_admin` for `sc_catalog` writes. Table: `sys_user`, `sys_user_role`, `sys_user_has_role`.
- **Service Catalog item** — the UC1 catalog item and its variables. Tables: `sc_cat_item`, `item_option_new`.
- **CMDB** — CI class usage and any seed CI records for UC3. Table: `cmdb_ci_server`.
- **Spoke connection configuration** — connection alias and credential record pointing at AAP.
- **Flow Designer flow** — *verify the automation path first.* Flow records (`sys_hub_flow` and related) are awkward to build field-by-field via REST. If direct creation proves impractical, ship the flow as an XML update set in `files/updatesets/` and have the playbook import and commit it. Either way it is automated — the update set is committed content, not a manual click-through.
- **Deprovision the seed account** as the final task.

### 0.3 AAP-side bootstrap (`bootstrap_aap.yml`)

Config-as-code via `ansible.controller`, `ansible.eda`, and `ansible.platform`:

- **Custom credential type** injecting `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` as env vars — the `servicenow.itsm` collection reads these automatically, so no playbook references a credential.
    - inputs: `sn_host`, `sn_username`, `sn_password` (secret)
    - injectors: the three env vars
- **ServiceNow credential** of that type
- **Project** pointing at this repo
- **Job templates** for each UC playbook, named distinctly from AIOps objects
- **OAuth application** for the Spoke to authenticate against (gateway object). Note: AAP 2.5+ auto-applies an API Script that causes 401s with the Spoke — the bootstrap must remove it, and the connection alias must use the `/controller/v2` endpoint path.
- **EDA**: decision environment, Event Stream, and the rulebook activation for `zabbix-to-snow.yml`

### 0.4 Zabbix-side bootstrap (`bootstrap_zabbix.yml`)

Via `community.zabbix`:

- **Trigger action** pointing at this demo's Event Stream, scoped to a host group or tag the AIOps demo does not use
- **Media type / webhook** as required by that action

Must be additive — read existing actions first and fail loudly rather than modifying anything the AIOps demo owns.

## Phase 1 — UC2: Zabbix → EDA → SNow incident

Starting with UC2 because it has no Spoke dependency — no IntegrationHub subscription, no Store install — so it is the fastest path to a working end-to-end flow.

### Components (all new, all in this subdir)

1. **Rulebook `extensions/eda/rulebooks/zabbix-to-snow.yml`** — own EDA rulebook with a webhook/Event Stream source receiving Zabbix alerts. Routes on event severity/tag to the incident job template. Independent of the AIOps demo's rulebook.

2. **Playbook `snow_create_incident.yml`** — creates a ServiceNow incident from the Zabbix payload:
   - Short description from Zabbix alert name
   - Description with host, alert details, timestamps
   - Caller: service account
   - Impact/urgency mapped from Zabbix severity
   - `set_stats` to pass `ticket_number` and `incident_sys_id` downstream

3. **Playbook `snow_update_incident.yml`** — updates/resolves the incident:
   - On recovery: resolve with work notes
   - On escalation: add work notes, leave open, bump priority

4. **Own rulebook activation + job templates** in AAP, named distinctly so they are not confused with the AIOps demo's objects.

5. **Zabbix side** — add a trigger action pointing at this demo's Event Stream, scoped so it does not overlap with the AIOps demo's existing action. Verify existing config first (see scope boundary above).

6. **CMDB update** — after incident resolution, update the CI record with current state

### Role: `servicenow_incident`

Dispatch on variable (`servicenow_action: create|update|close|find`):
- `create`: `servicenow.itsm.incident` with state=new
- `update`: add work notes, change state
- `close`: set state=resolved, close_code, close_notes
- `find`: `servicenow.itsm.incident_info` by number or sys_id

Uses only stock ServiceNow incident fields — no custom `u_*` columns, no hardcoded caller.

## Phase 2 — UC1: Service Catalog → Spoke → AAP

All setup for this UC is created by Phase 0 — the catalog item, Spoke connection, Flow Designer flow, and OAuth application. The only thing this phase adds is the fulfilment logic.

The one external dependency is the Spoke app itself being present from the ServiceNow Store (see Prerequisites) and the IntegrationHub subscription that gates it.

**Playbook `snow_catalog_fulfill.yml`** — launched by the Spoke with the catalog variables as extra_vars:
- Reads request parameters
- Does the provisioning work (simulated for demo purposes)
- Updates the RITM in ServiceNow (`state=closed_complete`)
- Creates/updates the CMDB CI record

### Role: `servicenow_catalog`

- Retrieve RITM details by sys_id
- Resolve variable mappings
- Close RITM with fulfillment notes

## Phase 3 — UC3: CMDB Reconciliation

### Playbook `snow_cmdb_reconcile.yml`

1. Gather facts from managed hosts (hostname, OS, IP, memory, CPUs)
2. Query ServiceNow CMDB for matching CIs (`servicenow.itsm.configuration_item_info`)
3. Compare gathered facts vs CMDB records
4. Report discrepancies
5. Optionally update CMDB to match reality (`servicenow.itsm.configuration_item`)

### Role: `servicenow_cmdb`

- `query`: find CI by name/IP/sys_id
- `update`: update CI fields (ip_address, os, host_name, fqdn, state)
- `create`: create new CI if host exists in inventory but not in CMDB
- `reconcile`: compare and report/fix

## Implementation order

1. **Phase 0** — bootstrap across ServiceNow, AAP, and Zabbix
2. **Phase 1 UC2** — incident management (no Spoke dependency, fastest end-to-end)
3. **Phase 2 UC1** — catalog/Spoke (gated on IntegrationHub + Store install)
4. **Phase 3 UC3** — CMDB reconciliation (most standalone, can be done last)

Build Phase 0 incrementally alongside the UCs rather than up front: add each object to the bootstrap as the UC that needs it is built, so the bootstrap is always exercised by something.

## Verification

- **Reproducibility (the headline test):** run `bootstrap.yml` against a fresh ServiceNow instance with nothing but the seed account, then run all three UCs. Any manual step discovered here is a bug in Phase 0.
- **Idempotence:** run `bootstrap.yml` twice — the second run reports no changes.
- **Teardown:** `teardown.yml` leaves the instance clean enough that `bootstrap.yml` succeeds again afterwards.
- **Seed deprovisioned:** confirm the seed account is inactive once bootstrap completes.
- **UC2:** Zabbix alert → this demo's EDA activation fires (not the AIOps one) → incident appears in ServiceNow → resolves on recovery
- **UC1:** catalog order → AAP job launches via Spoke → RITM closes
- **UC3:** reconciliation run → discrepancy report → CMDB updates
- **Non-interference:** after Phase 0, confirm the AIOps self-healing demo still runs unchanged
- **Credential injection:** smoke-test `servicenow.itsm.incident_info` to confirm the env-var injection works
