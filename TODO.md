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

- [ ] **`collections/requirements.yml` will break project sync.** At a repo
      root AAP runs `ansible-galaxy install` against it, and `community.zabbix`
      is neither in `ee-supported-rhel9` nor on the configured galaxy server.
      The file was inert when this was a subdirectory of the monorepo; the
      2026-10-05 split made it live. Either sync the collection to private
      automation hub, or move the file off the magic path — the orchestrator
      repo hit the same failure and took the second route.

      No AAP project exists for this repo yet, so nothing is currently broken.
      Resolve before creating one.

- [ ] **No `.gitignore` in this repo.** The umbrella's
      `collections/ansible_collections/` rule does not apply to submodules, so
      a local `ansible-galaxy collection install` would drop the installed tree
      into the working copy and offer it for commit.

## Phase 0 — bootstrap (not started)

Build these incrementally alongside the use case that needs them, rather than
all up front, so the bootstrap is always exercised by something.

- [ ] `bootstrap_servicenow.yml` — integration account and roles, catalog item
      and its variables, CMDB class usage, Spoke connection, Flow Designer
      flow, then seed account deprovision. Uses `servicenow.itsm.api` for
      generic table CRUD; the typed modules cover ITSM records but not platform
      configuration.
- [ ] **Open question: the Flow Designer flow.** Building `sys_hub_flow` and
      its related records field-by-field over REST may be impractical. The
      fallback is shipping the flow as an XML update set in `files/updatesets/`
      and having the playbook import and commit it — still automated, different
      mechanism. Verify which is viable before committing to an approach.
- [ ] `bootstrap_aap.yml` — credential type injecting `SN_HOST` / `SN_USERNAME`
      / `SN_PASSWORD` as env vars, the credential itself, project, job
      templates, OAuth application, EDA decision environment, event stream and
      rulebook activation.
- [ ] **AAP 2.5+ Spoke gotcha**, to handle in the AAP bootstrap: an
      auto-applied API Script causes `401`s with the Spoke and must be removed,
      and the connection alias must use the `/controller/v2` endpoint path.
- [ ] `bootstrap_zabbix.yml` — trigger action and media type via
      `community.zabbix`, scoped to its own host group. Must be additive: read
      existing actions first and fail loudly rather than modifying anything the
      AIOps demo owns. Blocked on the `community.zabbix` availability question
      above.
- [ ] `bootstrap.yml` and `teardown.yml` — entry points importing the three.

## Use cases (not started)

- [ ] **UC2 first** — no Spoke dependency, so fastest to a working flow.
      Rulebook `extensions/eda/rulebooks/zabbix-to-snow.yml`, plus
      `snow_create_incident.yml` and `snow_update_incident.yml`. Stock incident
      fields only: no custom `u_*` columns, no hardcoded caller.
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

The headline test for "everything automated": run `bootstrap.yml` against a
fresh ServiceNow instance holding nothing but the seed account, then run all
three use cases. Any manual step discovered there is a bug in Phase 0.

- [ ] Bootstrap is idempotent — a second run reports no changes
- [ ] `teardown.yml` leaves the instance clean enough that `bootstrap.yml`
      succeeds again
- [ ] Seed account is inactive once the bootstrap completes
- [ ] **Non-interference:** the AIOps self-healing demo still runs unchanged

## Context worth knowing

- The 2026-10-05 monorepo split squashed this project's history into a single
  `initial clean state` commit. Earlier commits exist only in the private
  archive, which is treated as forgotten and is not a content source.
- Collection audit of `ee-supported-rhel9` on 2026-10-04: `servicenow.itsm`
  2.16.0, `ansible.controller` 4.8.9, `ansible.eda` 2.12.1 and
  `ansible.platform` 2.7.20260812 are present; `community.zabbix` is not. The
  `servicenow.itsm` version clears the 2.12.0 floor its EDA source plugin needs.
