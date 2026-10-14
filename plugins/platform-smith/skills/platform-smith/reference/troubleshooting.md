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

1. **`list_integration_connections`** — does a `claude_code` / `codex` credential exist, and is it
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

**`get_agent_definition_resolved_files`** — the effective merged file set, which is what actually
lands in the container. Compare against `list_agent_definition_files` (what the definition itself
declares). Inheritance from an outer scope is almost always the difference.

## A published workflow runs but a node fails on credentials

**`get_workflow_definition_secret_refs_status`** (and `list_workflow_definition_secret_refs`) — the
workflow twin of the agent-definition check.

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
