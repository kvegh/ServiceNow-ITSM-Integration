# TODO

Open items for this repository. Nothing here has been run yet — the repo holds
a design plan and scaffolding only. See [PLAN.md](PLAN.md) for the full design
and [docs/00-prerequisites.md](docs/00-prerequisites.md) for what must exist
first.

## Next actionable step

- [ ] **Create `vars/vault.yml`.** Only `vars/vault.yml.example` is committed,
      so nothing can authenticate yet. The seed account exists and its
      credentials are available; they have not been put into a vault file.

          cp vars/vault.yml.example vars/vault.yml
          $EDITOR vars/vault.yml
          ansible-vault encrypt vars/vault.yml

      Confirm `$ANSIBLE_VAULT;1.1;AES256` is the first line with
      `git diff --staged vars/vault.yml` before committing.

- [ ] **Then: a connectivity test** against the instance using the seed
      credentials, before writing any bootstrap logic. Nothing has ever
      contacted the ServiceNow instance from this repo.

## Blocking an AAP project for this repo

- [ ] **Resolve project-root collection requirements before sync.** The plan
      uses a dedicated AAP project pointing at this standalone repository over
      HTTPS, so `collections/requirements.yml` is at its project root. Earlier
      notes recorded `community.zabbix` as missing from the selected EE and
      collection source. Verify the current source, make the collection
      available for project sync, and verify its Python dependencies in the EE.
      The file was inert when this was a subdirectory of the monorepo; the
      split changed that assumption.

      No AAP project exists for this repo yet, so nothing is currently broken.
      Resolve before creating one.

- [ ] **Use a dedicated Snow AAP project.** Replace the umbrella-project
      default in `vars/main.yml` before bootstrap. Job-template and rulebook
      paths must be relative to this standalone repository root.
- [x] **Local collection ignore rule.** This repository now ignores
      `collections/ansible_collections/`; umbrella ignore rules do not apply
      to submodules.

## Phase 0 — bootstrap (not started)

Build these incrementally alongside the use case that needs them, rather than
all up front, so the bootstrap is always exercised by something.

- [ ] `bootstrap_servicenow.yml` — integration account and roles, catalog item
      and its variables, CMDB class usage, Spoke connection, Flow Designer
      flow for the enabled use cases. Retain the setup account for reruns and
      teardown. Uses `servicenow.itsm.api` for
      generic table CRUD; the typed modules cover ITSM records but not platform
      configuration.
- [ ] **Open question: the Flow Designer flow.** Building `sys_hub_flow` and
      its related records field-by-field over REST may be impractical. The
      candidate fallback is an XML update set in `files/updatesets/`. Prove
      automated import, preview, commit, activation, and dependency handling
      before claiming this path works.
- [ ] **Verify integration-account roles and table permissions.** The scaffold's
      `rest_api_explorer` and `catalog_admin` roles do not establish incident or
      CMDB write access. Test incident read/create/update/resolve with the
      integration account, then request and CI operations as their UCs are added.
- [ ] `bootstrap_aap.yml` — credential type injecting `SN_HOST` / `SN_USERNAME`
      / `SN_PASSWORD` as env vars, the credential itself, project, job
      templates, OAuth application, EDA decision environment, event stream and
      rulebook activation.
- [ ] **UC1 feasibility and version compatibility.** Verify ServiceNow-to-AAP
      connectivity, TLS trust, Spoke availability, OAuth authorization/refresh,
      and Flow deployment. Document unavoidable manual prerequisites. Apply
      the reference's OAuth API Script workaround only if tested as necessary
      for our installed versions and demo-owned application registry. Verify
      the complete `/api/controller/v2/` endpoint and the Spoke's assembled URL.
      Keep UC1 components gated so they do not block UC2.
- [ ] `bootstrap_zabbix.yml` — trigger action and media type via
      `community.zabbix`, scoped to its own host group. Must be additive: read
      existing actions first and fail loudly rather than modifying anything the
      AIOps demo owns. Blocked on the `community.zabbix` availability question
      above.
- [ ] `bootstrap.yml` and `teardown.yml` — entry points importing the three.
- [ ] Optional `finalize_setup.yml` — deactivate the setup account only after
      verification and documentation of future authorized admin access.

## Use cases (not started)

- [ ] **UC2 first** — no Spoke dependency, so fastest to a working flow.
      Rulebook `extensions/eda/rulebooks/zabbix-to-snow.yml`, plus
      `snow_create_incident.yml` and `snow_update_incident.yml`. Stock incident
      fields only: no custom `u_*` columns, no hardcoded caller.
- [ ] **UC2 incident correlation.** Persist the original problem ID in the
      stock `correlation_id` field and look it up for independent recovery jobs.
      Repeated problem notifications reuse the same incident; repeated recovery
      leaves it resolved. Report missing or ambiguous matches clearly.
- [ ] **UC1** — `snow_catalog_fulfill.yml`, launched by the Spoke with catalog
      variables as extra vars; closes the RITM and updates the CMDB CI.
- [ ] **UC3** — `snow_cmdb_reconcile.yml`: gather host facts, query CMDB,
      compare, report, optionally correct.

## Docs to write as the code lands

Only `docs/00-prerequisites.md` exists. The rest describe playbooks that do not
yet exist and would be fiction today.

- [ ] `docs/01-bootstrap.md` — how to run the bootstrap, what it creates
- [ ] `docs/02-uc1-service-catalog.md`
- [ ] `docs/03-uc2-incident-management.md`
- [ ] `docs/04-uc3-cmdb-reconciliation.md`
- [ ] `docs/05-teardown.md`

## Verification targets

Run bootstrap for the enabled use cases against a clean test instance with the
documented prerequisites. Verify UC2 without Spoke setup; add UC1 after its
feasibility checks pass. Document unavoidable manual prerequisites explicitly.

- [ ] Bootstrap is idempotent — a second run reports no changes
- [ ] `teardown.yml` leaves the instance clean enough that `bootstrap.yml`
      succeeds again
- [ ] Bootstrap and teardown retain setup-account access; optional finalization
      has a documented way to obtain authorized admin access for future resets
- [ ] Integration-account permissions and injected credentials work for enabled UCs
- [ ] Repeated alerts and independently launched recovery update the same incident
- [ ] **Non-interference:** the AIOps self-healing demo still runs unchanged

## Context worth knowing

- The 2026-10-05 monorepo split squashed this project's history into a single
  `initial clean state` commit. Earlier commits exist only in the private
  archive, which is treated as forgotten and is not a content source.
- Collection audit of `ee-supported-rhel9` on 2026-10-04: `servicenow.itsm`
  2.16.0, `ansible.controller` 4.8.9, `ansible.eda` 2.12.1 and
  `ansible.platform` 2.7.20260812 are present; `community.zabbix` is not. The
  `servicenow.itsm` version clears the 2.12.0 floor its EDA source plugin needs.
