# ServiceNow ITSM Integration for AAP

## Context

A standalone ServiceNow ITSM integration demo in its own `ServiceNow-ITSM-Integration` repository, linked into AAP-advanced-features as a Git submodule, covering three use cases:

1. **UC1 — Service Catalog → Spoke → AAP:** ServiceNow catalog item triggers AAP job via Ansible Spoke, job processes the request and updates/closes the SNow ticket + CMDB
2. **UC2 — Zabbix → EDA → SNow incident:** Zabbix fires an event, EDA triggers an AAP job, Ansible creates/updates/closes a ServiceNow incident
3. **UC3 — CMDB reconciliation:** Playbook that compares infrastructure reality against ServiceNow CMDB and flags/fixes discrepancies

Uses the certified `servicenow.itsm` collection throughout.

### Automation goal and setup account

**Automate the demo's configuration.** Objects on ServiceNow, AAP, and Zabbix should be reproducible through Phase 0. Licensing and Spoke availability are prerequisites. UC1's OAuth authorization and Flow deployment must be proven before claiming fully automated setup; any unavoidable manual prerequisite must be documented explicitly.

The **seed service account** is the initial admin account used to create the first configuration, including a separate integration account for everyday demo jobs. It is created by hand because the bootstrap needs an existing API account to authenticate. Validate its API authentication under the instance's existing access policies.

Keep this setup account available for bootstrap reruns, development, and teardown. Bootstrap must not deactivate it automatically. Optional deactivation belongs in a separate finalization step, after successful verification and after documenting how authorized admin access will be restored or supplied for future setup and resets. Deactivation means making the account inactive, not deleting it.

Docs in `docs/` describe how to *run* the automation and what it creates — they are not click-through instructions.

### Own content only

All playbooks, roles, and rulebooks in this demo are written from scratch. The shadowman-lab reference repo (https://github.com/shadowman-lab/Ansible-SNOW) is **consultation material only** — read it when stuck on how a ServiceNow API or module behaves, do not copy or adapt its files. Rationale: its content carries instance-specific assumptions (custom `u_*` columns, hardcoded callers) and a structure built for a different demo; inheriting that imports debt we would have to unpick later.

### Scope boundary — independent of the AIOps demo

This demo is **separate from the AIOps self-healing demo** and must not modify it. The AIOps demo is already working and is duplicated across two locations (`aiops-selfhealing-demo` as its own public repo, and `AAP-advanced-features/AIOps_selfhealing_demo`), so edits there would need syncing in both and risk breaking a demo that currently runs.

UC2 reuses Zabbix as an event source, but with its **own** rulebook, own rulebook activation, and own job templates — all living in this subdir. No AIOps files, workflows, or AAP objects are touched.

**Shared-Zabbix caution:** the AIOps demo already has a Zabbix→EDA path configured. Before wiring UC2, verify what trigger actions / media types / Event Streams already exist so this demo adds its own rather than altering theirs. Prefer a distinct trigger action and a distinct Event Stream, scoped to a host group or tag that the AIOps demo does not use.

## Prerequisites

Start UC2 with the setup account, working API access, and its required collections. UC1 adds licensing, Spoke availability, and the feasibility checks below; these must not block the incident-management demo.

### Manual (unavoidable)

- [ ] **Seed/setup service account on ServiceNow** — admin-scoped user, Active=true, Locked out=false, **Web service access only=true**, Password needs reset=false. Verify API authentication under the instance's existing access policies. Retain access for bootstrap reruns and teardown; optional deactivation is a separate finalization step.
- [ ] **IntegrationHub subscription** — licensing gate for the Ansible Spoke (UC1). Cannot be automated; confirm it is active on the instance before planning on UC1.

### Verify before starting

- [ ] **AAP project source** — use a dedicated `SNOW Demo - Project` pointing at the standalone Snow repository over HTTPS. Job-template paths are relative to this repository, such as `playbooks/snow_create_incident.yml`; `collections/requirements.yml` is at its project root. During implementation, replace the scaffold's umbrella-project default in `vars/main.yml` before bootstrap. The umbrella repository remains a submodule index and is not this demo's AAP project source.
- [ ] **`servicenow.itsm` availability** — verify it in the chosen execution environment or use project-root `collections/requirements.yml` with an accessible, authorized collection source. Earlier availability notes are historical observations, not proof of the current environment.
- [ ] **Other collections** — `ansible.controller` and `ansible.eda` (AAP config-as-code), `ansible.platform` (gateway objects such as the OAuth application), `community.zabbix` (Zabbix trigger action). Confirm each is reachable from the chosen EE.
- [ ] **EE choice** — check whether `ee-supported-rhel9` carries the above. If not, prefer a project-level `collections/requirements.yml` over building a custom EE.
- [ ] **Zabbix bootstrap dependency** — earlier scaffold notes recorded `community.zabbix` as missing from the chosen EE. Resolve its availability before running Zabbix bootstrap and verify its required Python dependencies in that EE. A collection download alone does not establish that its runtime dependencies are present.
- [ ] **Integration-account permissions** — determine and document the roles and table permissions required by the enabled use cases. The scaffold's `rest_api_explorer` and `catalog_admin` list is not a verified incident/CMDB permission set. Verify incident read/create/update/resolve access for UC2, request access for UC1, and CI read/write access for UC3 using the integration account rather than the admin setup account.
- [ ] **Spoke app installation** — installing the Ansible Spoke from the ServiceNow Store is likely not API-automatable. Verify; if it is a one-time Store install, document it as a prerequisite rather than pretending the bootstrap covers it.
- [ ] **UC1 feasibility** — verify ServiceNow-to-AAP network reachability and TLS trust, the installed Spoke and AAP versions, OAuth token acquisition/refresh, and automated Flow deployment. Decide on a supported connection path, including a MID Server where required, without bypassing access controls. Document any unavoidable authorization step. Keep UC1 bootstrap components gated until these checks pass.

## Phase 0 — Bootstrap (everything as code)

Creates the enabled demo objects on all three systems. Driven by `bootstrap.yml`, which imports the per-system playbooks below. Build and verify UC2's bootstrap first; add gated UC1 and UC3 components alongside their use cases so an unavailable Spoke does not prevent UC2 setup.

### 0.1 Directory structure

```
ServiceNow-ITSM-Integration/
    PLAN.md
    README.md
    bootstrap.yml                         # one entry point: imports all three below
    teardown.yml                          # reverse, for a clean demo reset
    finalize_setup.yml                    # optional setup-account deactivation
    design_decisions_history.md
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

Runs as the setup account and creates the enabled ServiceNow objects. Uses `servicenow.itsm.api` for generic table CRUD, since the collection's typed modules cover ITSM records but not platform configuration.

- **Service accounts and roles** — create the demo's own integration user, separate from the setup account. Assign the roles and table permissions verified for the enabled use cases; do not assume REST API Explorer access grants incident or CMDB write access. Catalog configuration is created using the setup account. Tables: `sys_user`, `sys_user_role`, `sys_user_has_role`. Smoke-test the integration account before creating its AAP credential or treating bootstrap as successful.
- **Service Catalog item** — the UC1 catalog item and its variables. Tables: `sc_cat_item`, `item_option_new`.
- **CMDB** — CI class usage and any seed CI records for UC3. Table: `cmdb_ci_server`.
- **Spoke connection configuration** — connection alias and credential record pointing at AAP.
- **Flow Designer flow** — *prove the deployment path first.* Flow records (`sys_hub_flow` and related) are awkward to build field-by-field via REST. An XML update set in `files/updatesets/` is a candidate, but automated import, preview, commit, and flow activation must be verified with its dependencies. Do not claim automation merely because an XML file exists. Keep instance-specific credentials and endpoints out of committed update sets.
- **Retain the setup account** after bootstrap. Optional `finalize_setup.yml` deactivates it only after successful verification and a documented admin-access path for later bootstrap and teardown. Teardown does not deactivate the setup account.

### 0.3 AAP-side bootstrap (`bootstrap_aap.yml`)

Config-as-code via `ansible.controller`, `ansible.eda`, and `ansible.platform`:

- **Custom credential type** injecting `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` as env vars — the `servicenow.itsm` collection reads these automatically, so no playbook references a credential.
    - inputs: `sn_host`, `sn_username`, `sn_password` (secret)
    - injectors: the three env vars
- **ServiceNow credential** of that type
- **Dedicated project** pointing at the standalone Snow repository over HTTPS, with project-root collection requirements and repository-relative playbook/rulebook paths. Do not repurpose the shared umbrella project.
- **Job templates** for each UC playbook, named distinctly from AIOps objects
- **OAuth application** for the Spoke to authenticate against (gateway object), with an AAP identity authorized to execute this demo's templates. Verify authorization and refresh against the installed versions. The reference describes a ServiceNow-applied OAuth API Script causing a 401 with some AAP 2.5 setups; clearing that field is a conditional, tested compatibility change to this demo's own application registry, not an unconditional bootstrap action. For AAP 2.5+, verify the full controller API path `/api/controller/v2/`; the Spoke's version attribute may be `controller/v2` because its action supplies `/api/` itself. Test the assembled URL rather than treating the attribute as a complete endpoint.
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
   - Persist the original Zabbix problem event ID in the stock `correlation_id` field, with a demo/source prefix
   - Look up that correlation ID first: create when absent, otherwise reuse/update the existing incident so repeated problem notifications do not create duplicate tickets
   - `set_stats` may pass `ticket_number` and `incident_sys_id` within an AAP workflow; independent EDA launches must look up the persisted correlation ID

3. **Playbook `snow_update_incident.yml`** — updates/resolves the incident:
   - Recovery and escalation payloads carry the original problem event ID so each job can find the same incident without relying on a previous job's artifacts
   - On recovery: resolve with work notes
   - On escalation: add work notes, leave open, bump priority
   - Repeated recovery leaves an already resolved incident resolved; a missing or ambiguous match is reported clearly without updating an unrelated incident

4. **Own rulebook activation + job templates** in AAP, named distinctly so they are not confused with the AIOps demo's objects.

5. **Zabbix side** — add a trigger action pointing at this demo's Event Stream, scoped so it does not overlap with the AIOps demo's existing action. Verify existing config first (see scope boundary above).

6. **CMDB update** — after incident resolution, update the CI record with current state

### Role: `servicenow_incident`

Dispatch on variable (`servicenow_action: create|update|close|find`):
- `create`: `servicenow.itsm.incident` with state=new
- `update`: add work notes, change state
- `close`: set state=resolved, close_code, close_notes
- `find`: `servicenow.itsm.incident_info` by correlation ID for independent event handling, or by number/sys_id where already known

Uses only stock ServiceNow incident fields — no custom `u_*` columns, no hardcoded caller.

## Phase 2 — UC1: Service Catalog → Spoke → AAP

Add this UC's gated bootstrap components after its feasibility checks pass: the catalog item, Spoke connection, Flow Designer flow, and OAuth application. Then implement the fulfilment logic. Confirm a catalog order actually launches an AAP job before claiming fully automated UC1 setup.

External prerequisites include the installed Spoke, its licensing entitlement, and a working ServiceNow-to-AAP connection. Resolve OAuth authorization and Flow deployment as described in Prerequisites; document any unavoidable manual authorization or installation step rather than classifying it as an automation bug. UC2 remains runnable independently.

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

- **Reproducibility (the headline test):** run bootstrap for the enabled use cases against a clean test instance with the documented prerequisites, then run those UCs. Verify UC2 without UC1/Spoke setup. Once UC1's feasibility checks pass, repeat with all three UCs and confirm that every unavoidable manual prerequisite is documented.
- **Idempotence:** run `bootstrap.yml` twice — the second run reports no changes.
- **Teardown:** `teardown.yml` leaves the instance clean enough that `bootstrap.yml` succeeds again afterwards.
- **Setup-account lifecycle:** confirm bootstrap and teardown leave the setup account available. Test optional finalization only after verification and confirming the documented way to regain authorized admin access for future setup or resets.
- **UC2:** Zabbix alert → this demo's EDA activation fires (not the AIOps one) → incident appears in ServiceNow → resolves on recovery
- **UC2 correlation and repeats:** repeat a problem notification and confirm it reuses the same incident; launch recovery independently and confirm it resolves that incident; repeat recovery and confirm it stays resolved. Verify missing/ambiguous matches are reported clearly.
- **UC1:** catalog order → AAP job launches via Spoke → RITM closes
- **UC3:** reconciliation run → discrepancy report → CMDB updates
- **Non-interference:** after Phase 0, confirm the AIOps self-healing demo still runs unchanged
- **Integration-account access and credential injection:** use the integration account through the AAP credential to read, create, update, and resolve a disposable demo incident. Verify CI read/write access before UC3 and request access before UC1. Successful admin-account operations are not evidence that the integration account has these permissions.
- **Project and dependencies:** verify the dedicated project sync resolves repository-relative playbooks and project-root collection requirements, then smoke-test the required collections and Python dependencies in the selected EE before system bootstrap.

## Supporting references

- [ServiceNow REST API access and ACLs](https://www.servicenow.com/docs/r/api-reference/rest-api-explorer/c_RESTAPI.html)
- [ServiceNow Ansible Spoke setup](https://www.servicenow.com/docs/r/integrate-applications/integration-hub/setup-ansible.html)
- [Collection incident lookup and filters](https://github.com/ansible-collections/servicenow.itsm/blob/main/docs/servicenow.itsm.incident_info_module.rst)
- [Original consultation material](https://github.com/shadowman-lab/Ansible-SNOW) — reference only; no file copying or adaptation.
