# VM Scheduled Operations

**Status:** Draft for review.

| Field | Value |
|---|---|
| Author(s) | Eran Cohen, with AI assistance |
| Jira | [OSAC-4333](https://redhat.atlassian.net/browse/OSAC-4333) |
| Date | 2026-10-25 |

## Problem Statement

Tenant teams need VMs for known windows such as business hours, batch jobs, and demos. Starting and stopping them manually leads to missed starts, VMs left running after hours, and repeated operational work. Scheduling power-state changes lets teams plan VM availability without relying on someone remembering each operation.

Scheduling has no service charge. Resource metering and billing follow the usual OSAC policies, as for equivalent manual operations.

## In Scope

- **VMaaS only:** Create, view, update, and cancel schedules for Start or Stop of an existing VM. The underlying scheduling framework is generic, but V1 restricts exposed operations strictly to declarative power state changes (patching `run_strategy`). Exposing other fields requires extensive testing coverage and knowledge about what is actually supported, so additional operations are deferred to future features. Each schedule identifies one VM and one action. A user can create separate Start and Stop schedules for the same VM.
- **Timezones:** Recurring schedules use minute-resolution cron expressions. Users can specify an IANA timezone (e.g., `America/New_York`), which handles DST shifts automatically (skipping nonexistent times, executing once on repeated times). Use the first occurrence when a local time repeats. Execution timestamps and underlying execution/audit records remain in UTC; schedule definitions retain the selected IANA timezone. Invalid expressions and timezone names are rejected, and users can see the next scheduled occurrence.
- **Scheduled changes:** each occurrence requests the selected power state. The scheduler does not continuously enforce a time window. A manual power-state change remains effective until the next scheduled occurrence. For example, manually stopping a VM after its scheduled Start leaves it stopped until its next scheduled Start.
- **Permissions:** Tenant Users manage their own schedules for VMs they are allowed to operate. Tenant Admins can inspect, edit, or cancel schedules within their tenant. Creating or editing a schedule requires permission for its VM action, and each occurrence checks the schedule owner's current permissions. An Admin edit retains the owner and records who changed the schedule; it does not grant the owner new permissions. Owners can cancel their schedules even after losing permission to operate the VM.
- **Execution results:** users can view execution status, including success, failure, and missed occurrences, through the UI or API, and distinguish a pending power-state change from a completed or failed operation. A VM already in the requested state is a successful no-op. A deleted VM or lost permission is reported as a failure. Tenant Admins can inspect execution status within their tenant. Active push notifications, including UI alerts, and emails are deferred to V2 as a fast follow; V1 is not blocked on OSAC-75.
- **Missed or failed occurrences:** report the failure and leave future occurrences enabled. V1 does not automatically retry failed occurrences or replay missed ones after recovery. An older occurrence must not override a newer scheduled or manual change.
- **Conflicts:** If a Start and a Stop occurrence for the same VM trigger in the exact same scheduled minute, the system skips both and reports the conflict in the UI/API execution status. Conflicts are evaluated at execution time, before applying either change; V1 does not evaluate future cron intersections at schedule creation or update. Compare scheduled minutes in UTC after timezone conversion, rather than actual worker start times. Adjacent-minute actions are allowed with no minimum separation and are applied in scheduled-time order.
- **Reusable scheduling:** the underlying framework schedules generic JSON patches, with V1 exposing only the `run_strategy` power-state changes described above. Scheduled changes use the normal resource authorization and reconciliation path. V1 does not introduce dedicated imperative power-management APIs or restrict unrelated manual VM updates. The detailed patch representation and safe field allowlist are specified in the technical design.

## Out of Scope

- VM creation and deletion are explicitly excluded from V1. While the scheduler architecture is generic, these operations carry different safety semantics and require separate testing coverage that will be evaluated in future iterations. Creation/provisioning is excluded due to API complexity, and deletion/de-provisioning is deferred to keep this increment focused on power states.
- Resizing, changing instance types, and scaling of any kind, including time-based and usage-based scaling.
- Project, tag, owner/user, or other dynamic target selectors; bulk or multi-VM schedules.
- Scheduling updates to VM fields other than `run_strategy`, arbitrary API execution, and user-supplied patches beyond the V1 allowlist.
- Scheduling for clusters, bare metal, or other services.
- Rate-based and one-time schedules, and continuous enforcement of power-state windows.
- Configurable retries and catch-up execution.
- Active push notifications, UI alerts, and email notifications; these are deferred to V2. Viewing execution status in the UI/API remains in V1.
- Changes to Cloud Provider Admin or Cloud Infrastructure Admin capabilities.

## User Stories

### Tenant User

- As a Tenant User, I want to create, update, view, and cancel cron schedules in my chosen IANA timezone for Start or Stop of one existing VM I am allowed to operate, so that planned usage stays aligned with local time across DST changes.
- As a Tenant User, I want to see successful, failed, and missed occurrences in the UI or API, so that I can investigate when a scheduled change does not complete.

### Tenant Admin

- As a Tenant Admin, I want the same scheduling capabilities for VMs I am allowed to operate and the ability to inspect, edit, or cancel Tenant User schedules within my tenant, so that I can enforce tenant policy and manage operational problems.

## Dependencies

- UX and UI must be delivered through the bootstrap UX/UI Design gates. Completion requires the PRD and technical Design to be merged.

---

## Provenance

Authored: commit @ prd 0.11.3 - d9d66a5, workspace main @ 0fe0e4595 (15 behind origin/main)
Final: revise @ prd 0.11.3 - d9d66a5, workspace main @ a6d3626e5

> Context changed between commit and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.3","ai_workflows":"d9d66a5","source_repo":"a6d3626e5","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["commit","respond","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":true} -->
