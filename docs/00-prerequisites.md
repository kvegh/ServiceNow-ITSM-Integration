# 00 — Prerequisites

Everything this demo needs that the bootstrap cannot create for itself. Each
item is either a licensing gate, a chicken-and-egg problem, or something to
verify before relying on it.

## Manual, unavoidable

### Seed service account

An admin-scoped ServiceNow user used for the first API call. There is no way to
automate this: nothing can authenticate to the API before an API account exists.

Required settings:

| Field | Value |
|---|---|
| Active | true |
| Locked out | false |
| **Web service access only** | **true** |
| Password needs reset | false |

The "Web service access only" flag is the one that matters. Without it, SSO/MFA
enforcement applies to the account and the API returns `401` regardless of the
password being correct.

The bootstrap creates its own integration account and deprovisions this seed
account as its final task.

### IntegrationHub subscription

Licensing gate for the Ansible Spoke, which UC1 depends on. Cannot be automated.
Confirm it is active before planning on UC1 — UC2 and UC3 do not need it.

### Spoke application install

Installing the Ansible Spoke from the ServiceNow Store is most likely not
API-automatable. Treat it as a one-time prerequisite rather than assuming the
bootstrap covers it. Verify before committing to either position.

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

Four of the five ship in the image, so AAP needs no install step for them. The
`servicenow.itsm` version also clears the 2.12.0 floor required by its EDA source
plugin.

`community.zabbix` is the gap. It is needed only by the Zabbix-side bootstrap
(Phase 0.4). Options: sync it to private automation hub, bake it into a custom
execution image, or perform that one bootstrap step by another means.

Re-check with:

    podman run --rm <ee-supported-image> ansible-galaxy collection list

### collections/requirements.yml is a trap at a repo root

This file is for local development only:

    ansible-galaxy collection install -r collections/requirements.yml

Note that AAP resolves `collections/requirements.yml` **at the project root**.
Before the monorepo split this file sat in a subdirectory and was inert. Now
that this repo is its own project root, AAP would run `ansible-galaxy install`
against it during project sync — and fail, because `community.zabbix` is neither
in the execution image nor on the configured galaxy server.

Either sync that collection, or move the file off the magic path, before
creating an AAP project for this repo. No project exists yet, so nothing is
currently broken.

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
