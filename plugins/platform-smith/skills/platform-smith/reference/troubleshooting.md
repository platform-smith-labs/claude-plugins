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
2. **`list_workspace_integrations`** — a credential that exists company-wide but is **not attached to
   this workspace** will not resolve for its runtimes.
3. **`get_agent_definition_secret_refs_status`** — do the refs actually resolve?
4. ⚠️ **Credentials are frozen into the runtime instance at launch.** A credential fixed (or rotated
   with `update_secret`) afterwards needs a **fresh runtime** — restarting the existing one changes
   nothing.

## Working tree is empty or not what you expected

1. **`get_project_git_link`** — is the project bound to a repo and branch at all? This is the single
   most useful check here.
2. **`list_git_connections`** — is a provider connected?
3. **`list_git_connection_repos`** — can the connection **see** that repo? If it is missing here, the
   installation does not cover it, and no amount of project config will fix that.

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
