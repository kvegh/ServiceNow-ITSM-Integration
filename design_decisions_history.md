# Design decisions history

## Initial plan and configuration scaffold — 2026-10-05

Commit: `3b0b84ed98a39aa0b8118e1c76af688882e858c6` (`initial clean state`).

This repository's initial commit introduced a three-use-case ServiceNow/AAP plan,
collection requirements, default variables, and a placeholder credential template.
The plan chose to build incident management first because it avoids the catalog
Spoke dependency. It also required independent objects to protect the working
AIOps demo and original content to avoid importing the reference repository's
instance-specific assumptions. These reasons are documented in the committed
plan.

Earlier design discussions and implementation history cannot be reconstructed
from this repository's initial commit; no additional rationale is inferred here.

## Project handoff documentation — 2026-10-13

Commit: `4938b13c6f5f3a4f2f7496ea0efee0d8ee6b27ff` (`add README, TODO and
prerequisites so the project can be picked up cold`).

Added a README, actionable task list, prerequisite guide, and a local collection
ignore rule so another contributor could understand the design and begin work.
The handoff also recorded a collection-availability concern after the repository
became a standalone project root. The handoff purpose is documented by the commit
subject; the collection concern is documented in its prerequisite guide. These
were planning and documentation changes, not implemented or deployed automation.

## Review corrections for repeatable demo execution — 2026-10-13

Implementation commit: `8feca9e7c5873a8a00a5c7798371b1fe060d7970` (`Refine Snow demo setup and incident execution plan`).

The user approved five focused corrections to the existing plan. Setup-account
deactivation became a separate optional final step so bootstrap reruns and
teardown retain admin access. Incident handling gained a persistent correlation
key so independently launched recovery jobs find the original incident and
repeated alerts reuse it. Integration-account permissions now require verification
for the actual incident, request, and CI operations rather than relying on the
initial role list.

The plan also requires proving Spoke connectivity, OAuth, and automated Flow
deployment before claiming complete UC1 automation, while keeping UC2 independent.
Its project source and collection requirements now align with a dedicated AAP
project for the standalone Snow repository. Dependency availability and Python
requirements must be verified before bootstrap. README, tasks, prerequisite
guidance, and scaffold comments were reconciled with these decisions.

These reasons come from the review and the user's explicit instruction to add
the changes. No automation was implemented or run, and no live accounts or
integration objects were changed by this documentation update.
