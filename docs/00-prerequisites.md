# 00 — Prerequisites

Everything this demo needs that the bootstrap cannot create for itself. Each
item is either a licensing gate, a chicken-and-egg problem, or something to
verify before relying on it.

## Manual, unavoidable

### Seed service account

The initial admin setup account used for the first API call and bootstrap.
Bootstrap needs an existing API account to authenticate before it can create
the separate integration account used for everyday demo jobs.

Required settings:

| Field | Value |
|---|---|
| Active | true |
| Locked out | false |
| **Web service access only** | **true** |
| Password needs reset | false |

Verify API authentication under the instance's existing access policies; do not
assume a single account flag guarantees successful authentication.

Bootstrap creates its own integration account and retains setup-account access
for reruns, development, and teardown. Deactivation is a separate optional
finalization step after verification, with a documented way to obtain authorized
admin access for future setup or resets. Teardown retains the setup account too.

### IntegrationHub subscription

Licensing gate for the Ansible Spoke, which UC1 depends on. Cannot be automated.
Confirm it is active before planning on UC1 — UC2 and UC3 do not need it.

### Spoke application install

Installing the Ansible Spoke from the ServiceNow Store is most likely not
API-automatable. Treat it as a one-time prerequisite rather than assuming the
bootstrap covers it. Verify before committing to either position.

### UC1 connection and deployment checks

Verify ServiceNow-to-AAP reachability and TLS trust, the installed Spoke/AAP
versions, OAuth authorization and token refresh, and automated Flow deployment.
Use a supported connection path, including a MID Server where needed. An XML
update set is a candidate deployment mechanism; automated import, preview,
commit, and activation must be demonstrated with its dependencies. Document any
unavoidable manual authorization step. These checks gate UC1, while UC2 remains
runnable independently.

The reference's OAuth API Script workaround is conditional on the installed
versions and a demonstrated need, and applies only to this demo's application
registry. Verify the assembled controller URL: `/api/controller/v2/` is the full
path, while `controller/v2` may be the Spoke's version attribute.

## Verify before starting

### Collections in the execution image

Audited in `ee-supported-rhel9` on 2026-10-04:

| Collection | Version | Present |
|---|---|---|
| `servicenow.itsm` | 2.16.0 | yes |
| `ansible.controller` | 4.8.9 | yes |
| `ansible.eda` | 2.12.1 | yes |
| `ansible.platform` | 2.7.20260812 | yes |
| `community.zabbix` | — | **no** |

This is a historical audit; recheck the currently selected image before relying
on it. Four of the five were present then. The
`servicenow.itsm` version also clears the 2.12.0 floor required by its EDA source
plugin.

`community.zabbix` was the gap. It is needed by the Zabbix-side bootstrap
(Phase 0.4). Resolve its collection-source availability and verify required
Python dependencies in the selected execution image before bootstrap.

Re-check with:

    podman run --rm <ee-supported-image> ansible-galaxy collection list

### Dedicated project and collection requirements

Use a dedicated `SNOW Demo - Project` that syncs this standalone repository over
HTTPS. Replace the scaffold's umbrella-project default in `vars/main.yml` during
implementation, before bootstrap runs. Playbook and rulebook paths are relative
to this repository root.

The project-root collection requirements also support local development:

    ansible-galaxy collection install -r collections/requirements.yml

Note that AAP resolves `collections/requirements.yml` **at the project root**.
Before the monorepo split this file sat in a subdirectory and was inert. Now
that this repo is its own project root, AAP resolves the file during project sync
when collection downloads are enabled. Earlier notes recorded `community.zabbix`
as unavailable from the configured source; an unavailable required collection
would cause sync to fail even if other required collections already exist in the EE.

Verify the source and make all declared collections available before project
sync. A collection download alone does not supply its Python dependencies.

### Integration-account permissions

Determine the roles and table permissions required by the enabled use cases.
The scaffold's `rest_api_explorer` and `catalog_admin` list is not a verified
incident/CMDB permission set. Use the integration account to test incident
read/create/update/resolve for UC2, request access before UC1, and CI read/write
before UC3. Admin-account success does not verify integration-account access.

## Vault

Secrets live in an ansible-vault encrypted `vars/vault.yml`. Encrypted vault
files are committed deliberately — the platform decrypts them at job runtime.

    cp vars/vault.yml.example vars/vault.yml
    $EDITOR vars/vault.yml
    ansible-vault encrypt vars/vault.yml

Before committing, confirm the file is actually encrypted:

    git diff --staged vars/vault.yml

The first line must read `$ANSIBLE_VAULT;1.1;AES256`. A vault credential on a
job template supplies only the decryption password — not the file, and not the
variables inside it.
