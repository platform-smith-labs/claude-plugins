# Troubleshooting

Work down the list. Each step names the tool that answers it. **Do not retry a failed launch until
one of them explains the failure** — a retried credential refusal fails identically.

## Launch never starts / stays queued

1. **`get_readiness`** — pass whichever of `workspace_uuid` / `environment_uuid` / `project_uuid` you
   have. This is the pre-flight; call it first when a launch fails for no obvious reason.
2. **`get_workspace_controllers_summary`** — **a disconnected controller looks exactly like a stuck
   launch.** If nothing is connected, there is nothing wrong with your request.
3. **`get_task`** on the `task_uuid` the launch returned; **`get_task_responses`** for what it
   produced.

## Start was refused for a missing coding-agent credential

Correct behaviour, not a fault. Branch on `error.detail`
(`coding_agent_credential_unavailable` / `credential_unusable`), **never** on `error.code`. Fix the
credential or pick another definition — see the pre-flight in `running-work.md`.

## Runtime starts, but the coding agent cannot authenticate

1. **`list_integration_connections`** — does a `claude_code` / `codex_cli` credential exist, and is it
   enabled? `get_integration_connection` for one in detail.
2. **`list_workspace_integrations`** — returns the `connections` and `accounts` **assigned to this
   workspace**. A credential that exists company-wide but is not assigned here will not resolve for
   its runtimes.
3. **`get_agent_definition_secret_refs_status`** — do the refs actually resolve?
4. ⚠️ **Credentials are frozen into the runtime instance at launch.** A credential fixed (or rotated
   with `update_secret`) afterwards needs a **fresh runtime** — restarting the existing one changes
   nothing.

## Working tree is empty or not what you expected

1. **`get_project_git_link`** — is the project bound to a repo and branch at all? This is the single
   most useful check here. It also names the **integration account** that authorizes the clone
   (`integration_account_uuid`, `account_login`, `provider_code`) — or none, for a public repo.
2. **`get_integration_account`** on that account — is it `suspended`, and is it **assigned to this
   workspace** (`assignment`)? A link whose account is suspended or not assigned here will not mint
   a clone token.
3. **`list_integration_account_repos`** — can the account **see** that repo? Search with `q` before
   concluding it is absent. If it is missing, that installation does not cover it, and no amount of
   project config will fix that. A `409 git_connection_reauth_required` means a person must renew
   the account's authorization in the app.

GitHub and Bitbucket are **integrations** now: each installation or workspace is an **account**
under an integration connection (`list_integration_connections` lists them with their accounts).

## Why didn't my automation fire?

1. **`list_automation_deliveries`** for that automation — one envelope-only row per event, with
   the outcome and the reason. Most answers are here: `disabled`, `paused`, `filtered`,
   `unknown_command` (a verb nothing answers — check it, and the prefix with
   `get_workspace_command_prefix`), `author_not_linked`, `built_in_off` (Slack @mentions and
   `/smith` switched off in this workspace), `no_credential`. Tracker labels and assignments no longer
   start anything (they only capture the ticket, with no delivery logged): use a command. The full table is in
   `automations.md`.
2. **`get_automation`** — is it `enabled`? Is it auto-paused (`pause`)? Does `source` say what you
   think (events, `command_verb`, the tracker container), and is the target still there?
3. **The event never appears in the log at all?** Then it never reached this automation:
   - **`get_integration_account`** — is **Receives events** on (`receives_events`)? New GitHub and
     Bitbucket accounts start with it **off**, and nothing fires from an account until a person
     turns it on in the app. Is the account `suspended`, or not assigned to the automation's
     workspace?
   - For a repository source: is the project still linked to that repository
     (`get_project_git_link`), and is the platform's App installed on it?
   - A webhook delivery with a bad signature or an unknown address is rejected before it is logged.
4. **A command did nothing?** The person must type the workspace's **effective** prefix
   (`get_workspace_command_prefix`) and then the verb, at the start of the comment — the prefix is
   case-sensitive.

## Who did my automation run as?

1. **`list_automation_runs`** — each run carries `identity` (`person` or `service_principal`),
   `acted_as`, and an `identity_note` when a person asked but the run fell back to the service
   principal:
   - `person_not_linked` — their provider account is not linked to a PlatformSmith user.
     **`list_integration_account_people`** shows who is linked (an org admin sees every link;
     anyone else only their own). Linking happens in the app.
   - `person_not_member` — linked, but not allowed to start that target in this workspace.
   - `person_unchecked` — the permission could not be checked at that moment; usually transient.
   - `person_target_unsupported` — that target does not act as a person for that kind of event.
2. **`get_automation`** → `identity` says what to expect: `service_principal` (always — schedules,
   webhooks, Slack alerts, pushes) or `invoker_when_linked` with the `invoker` (`pr_author`,
   `commenter`, `mentioner`). `run_automation_now` always runs as the service principal.
3. A Slack @mention or `/smith` never falls back to the service principal: if the sender could
   not act as themselves, nothing started and the delivery log says why.

A provider account can belong to only **one** organization: if a connect in the app reports the
account is claimed elsewhere, another organization holds it, and nothing on this side changes that.

## The runtime does not behave like its definition says

**`get_session_snapshot`** — what that session's agent **actually received**: the agent definition and
the version it ran, how it was chosen (a mandatory definition overrides an explicit pick), the sandbox
profile and its version, and the file list; **`get_session_snapshot_file`** reads one file. Nothing is
merged at launch, so if the snapshot differs from the definition today, the definition **changed since
the session started** — `compare_agent_definition_versions` shows what. Files are stored before secret
substitution: a file marked `has_secret_placeholders` shows the placeholder, never the value. A session
may have no snapshot (recording is best effort, and older sessions predate it).

A session can have **several snapshots** (for example a resume or a restart). Both tools take an optional
`snapshot` (a snapshot id from `snapshots[]`); leave it out for the latest. Each entry in `snapshots[]`
carries a `spawn_reason`. The snapshot also records the `inputs` the session received and a `runtime`
summary of what the sandbox reported; if `runtime_reported_at` is null the sandbox did not report. A
`snapshot_failed` marker means recording failed for that spawn. Read `inputs/...` and `runtime/...` files
with `get_session_snapshot_file` (its `source` is `definition`, `platform`, `endpoint`, `input` or
`runtime`). Secret refs show `delivered`, `delivery` and `value_version`, never a value. A 404 means there
is no such snapshot or recording failed. To see what changed between two snapshots (of one session or
two), use `compare_session_snapshots` with both session and snapshot ids.

A copy that "should have" picked up an organization change has not, unless someone applied it:
`get_agent_definition_update` on the copy shows what is waiting. Only **published** versions count.

## A published workflow runs but a node fails on credentials

**`get_workflow_definition_secret_refs_status`** (and `list_workflow_definition_secret_refs`) — the
workflow twin of the agent-definition check.

## A component would not attach, update or remove

Read `error.detail` (and `error.details`) and branch on it:

| `detail` | Meaning and next step |
|---|---|
| `harness_not_supported` (422) | Library **install** only: the item does not support the requested coding agent. Final: do not retry. Name the harness and offer another listing. Attach never refuses on this: it succeeds with a `component_not_read` warning. |
| `entry_owned` | Another component owns that entry. Detach or remove the other component; do not take its key. |
| `unsupported_layout` | The agent's files do not have the shape the component expects. Tell the person; do not force it. |
| `decisions_required` / `acknowledgement_required` | The person must decide each clash and acknowledge what it executes or connects to. Ask; never fill these in. |
| `plan_stale` / `stale_base_version` | The agent or component changed after the plan. Plan again and show it again. |
| `component_not_read` (a **warning**, not an error) | The write succeeded, but a component's listing does not support the named `harness`, so that coding agent will not read it. Tell the person; detach the component or keep it. |
| `skills_hidden_for_harness` (a **warning** on a save, a 422 at launch) | A playbook's skill is not visible to the harness; `path` is the skill folder. Add a folder link at that harness's skill root, or launch with a harness that sees it. |
| `home_path_not_allowed` (400) | A `~/` file path outside `~/.claude/` and exactly `~/.codex/config.toml`. Final: use a workspace-relative path. |
| `not_installable` | A component is attached, not installed: use `plan_attach_agent_definition_component`. |
| `source_ambiguous` / `source_required` | Attach and plan take exactly one of `library_code` or `org_component_uuid`. Send one (`source_required` = neither sent, `source_ambiguous` = both sent). |
| `item_not_found` | The component does not exist or this agent cannot see it (outside its scope). Check `list_available_agent_definition_components`. |
| `unpublished` (409) | Attach or plan found a component your organization has since unpublished. Final: pick another. |
| `org_component_unpublished` | Your organization unpublished it. Final: it cannot be added or published to again. |
| `invalid_component` (400) | Publishing checks failed. `error.details.problems` lists each one with a `message` and the `path` or `entry_key`; show them all, fix, and repeat with `dry_run: true`. Unknown rules are still problems. |
| `code_reserved` / `code_exists` (409) | The code belongs to PlatformSmith, or your organization already used it (even unpublished). Choose another code. |
| `stale_base_version` (409) | Someone published, or the agent changed, since you read it. Re-read, show the person again, then repeat. |
| `scope_ambiguous` / `scope_excludes_source` (400) | Send at most one of `workspace_uuid` / `project_uuid`; to keep tracking, publish where the source agent can see it. |
| `empty_selection` (400) | A draft needs at least one file or entry. |
| `confirm_required` (400) | `unpublish_org_component` needs `confirm: true` after the person agrees. |

## A workflow copy or Library workflow install was refused

Branch on the string code in `error.detail`, never on the numeric `error.code`; lists are in `error.details`.

| `detail` | Meaning and next step |
|---|---|
| `read_only_copy` (409) | A use-as-is or parent-locked copy: its graph can't be edited or restored. Workflows can't be customised in place. Fork to edit (`fork_workflow_definition`, after asking) — unless `read_only_reason` is `parent_locked`, which only the parent's owner can lift. |
| `not_published` (409) | The workflow you are copying has no published version. Ask its owner to publish first. |
| `dependency_missing` (409) | The update needs agents not installed here (`error.details.needs_install`). Install them from the Library, then apply again. |
| `task_type_unavailable` (422) | The workflow uses a step type that isn't available here (`error.details.task_types`). It can't be installed; do not retry. |
| `already_installed` (409) | A copy of this Library workflow is already at the target. Use that copy. |
| `workflow_service_unavailable` (502/503) | The workflow service did not answer. After an install, `error.details.completed: ["dependencies"]` means the agents were kept: plan again and install again. |
| `kind_mismatch` (400) | `kind` was missing or did not match the item. Send the item's `kind` exactly as the list shows it. |
| `catalogue_unavailable` (502) | The Library could not be reached for an update. Try again later; the copy keeps working. |
| `invalid_mode` (400) | `mode` must be `use_as_is` or `fork`. |

## A tool returned 403

**`get_my_permissions`** — read your own limits, rather than discovering them by failing more calls.

## A tool returned 404

Two very different causes, deliberately indistinguishable:

- The thing does not exist, **or**
- it belongs to another company. The platform will not confirm which. This is the cross-tenant
  non-leak working as designed — not something to retry or route around.

## An A2A message may not have landed

**`list_conversation_delivery_receipts`** distinguishes "posted" from "received" — the difference
between waiting patiently and waiting forever. `list_conversation_messages` shows what was actually
said; `list_conversation_sessions` bridges from what was said to the session that said it.

## Reconstructing what happened

**`get_workspace_audit`** — the workspace's audit trail, when session events and run histories each
show only their own slice.

## Before rotating a secret

**`get_secret_descendants`** — what inherits from it down the scope chain, so you know what else
moves. Then `update_secret` (rotation) rather than creating a second secret with a new name. Remember
the freeze rule above: running runtimes keep the old value.
