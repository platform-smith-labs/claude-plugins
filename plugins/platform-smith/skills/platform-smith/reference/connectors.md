# Connectors, guardrails and the tool audit

A **connector** is a vendor's MCP server — Jira and Confluence, Linear, GitHub, Sentry, Datadog,
Supabase, Context7, dbt — whose tools a coding-agent **session** can call *through* PlatformSmith.
The built-in connector `platform` holds the platform's own agent tools (work items, artifacts,
notifications, …). Connectors are **not integrations**: an integration
(`list_integration_connections`) is a credential the platform itself uses for git, trackers, Slack
and coding agents. A connector account is used only when an agent calls a connector tool.

## How a session gets a tool

Every gate below must say yes, and each one can only **narrow** what the one before allowed:

1. **The tools file lists it.** The agent definition's `.platform-smith/tools.json` names the
   connectors and platform families its sessions may use. No file means every platform tool and no
   connector tool.
2. **The organisation enabled the connector and reviewed the tool.** A tool nobody has reviewed is
   blocked for the whole organisation (`list_connector_tools` shows `reviewed`).
3. **No guardrail blocks it.** A guardrail blocks a tool, or makes each call wait for a person's
   approval, at company, workspace, project or environment level. **Guardrails only restrict** —
   they never grant what the tools file did not list — and the **strictest** matching rule wins.
4. **Wiring picks an account.** In order: the environment's override for that connector → the
   workspace's default account → the only account assigned to the workspace. Otherwise the tool is
   left out.

```json
{
  "platform":   { "off": ["work_items"] },
  "connectors": {
    "jira":          { "effects": ["read", "write"], "approval": "ask",
                       "overrides": { "jira_transition_issue": "ask" } },
    "jira[support]": { "effects": ["read"] }
  }
}
```

A connector entry defaults to `effects: ["read"]` and `approval: "ask"`. `jira[support]` addresses
the account labelled `support` in that workspace.

**Check before you save:** `preview_session_tools` answers what a session would get in a given
workspace / project / environment with a given tools file — granted tools, `left_out` with a reason
for each, the account each connector would act through, and which tools will ask first. It writes
nothing.

## "Why can't my agent call X?"

| Step | Tool |
|---|---|
| What did this session get at launch, and what was left out? | `get_session_tools` |
| What did it call, and what was decided? | `list_tool_audit` (filter by `session_uuid`) |
| Why exactly was one call blocked, held or allowed? | `explain_tool_decision` |
| Which rules reach this place? | `list_guardrails` with `workspace_uuid` (+ project / environment) |
| Is the tool reviewed? Does it need review after a vendor change? | `list_connector_tools` |
| Which accounts exist, and where are they assigned? | `list_connector_accounts` |

Common reasons a tool is left out or refused: `not_granted` (called, but not granted to this
session — usually the tools file does not list it), `not_reviewed`, `blocked_by_guardrail`, `connector_not_enabled`, `no_account_assigned`,
`no_default_account` (several accounts reach the workspace and none is its default),
`account_needs_reauthorization`, `binding_empty` (a parameter the platform fills from the session —
such as the work item's ticket — had nothing to fill it with). The granted set is fixed at launch, so
a tools-file change applies to the **next** session; guardrails are checked again on every call.

**The audit is metadata only.** A row records the tool, the decision, the reason, the rules, the
account and timings — never the call's arguments or its result. Organisation admins see every
workspace; workspace admins see their own.

## Admin actions

Every write tool here is an administrative action, and the platform enforces who may make it:
organisation admins for connectors and tool reviews; workspace admins for guardrails and
environment overrides inside their own workspaces. **Ask the person before calling
one**, and show them `preview_session_tools` first where it applies.

- `enable_connector` — create the organisation's copy of a catalog connector. Its tools arrive
  **unreviewed**, so enabling grants nothing by itself.
- `review_connector_tool` — the organisation-wide switch for one tool, and its effect class (equal
  to the catalog's or stricter). Turning a write tool on lets agents change things in the vendor's
  system.
- `create_guardrail`, `update_guardrail`, `delete_guardrail` — guardrails are versioned; every
  change is a new revision (`list_guardrail_revisions`). Updating or deleting one can **loosen** a
  restriction for every session it reached.
- `set_environment_connector_override`, `clear_environment_connector_override` — make an
  environment use a specific account (for example a read-only bot account in production).
- `discover_connector_tools`, `update_connector_tool` — the organisation's own servers, aggregators
  and the descriptor editor (below). Discovery and the editor change what agents may send to a server
  the platform has not reviewed; say so when proposing them.

## Custom connectors and the descriptor editor

A **custom connector** is any MCP server the organisation runs, added by an org admin by its address
in the PlatformSmith UI (Org → Connectors), which is also where it is edited or deleted. PlatformSmith
has not reviewed its tools: `discover_connector_tools` reads them from the server through one of its
accounts and makes one descriptor per tool (effect from the server's annotations, every input
agent-supplied, each **unreviewed** until `review_connector_tool`). Re-running discovery keeps the
organisation's edits for tools that still exist, retires tools the server dropped, and adds new ones
switched off. `list_custom_connectors` lists them.

The **descriptor editor** is `get_connector_tool` + `update_connector_tool`: read the tool (its
`version` is the save's `base_version`), then save a new version with the description, effect, every
parameter as `{from: "agent", …constraints}` or `{bind: "<session source>"}`, the result cap and the
timeout. A pinned parameter never reaches the agent. Sessions already running keep the version they
were launched with; the next launch uses the new one. A **catalog** connector's tool may only be
tightened (stricter effect, constraints added, agent → pinned); a custom tool may be shaped freely but
cannot invent an input the server does not have. Refusals name the field (`invalid_descriptor`).

Connecting an account to a custom connector is still done by a person in the UI (a pasted key, or a
sign-in that may ask for the organisation's own OAuth app when the server offers no self-registration).

## Aggregators

An **aggregator** (category aggregator; Composio is the first) is one vendor fronting many apps —
GitHub, Slack, Gmail, Notion, … — through the **organisation's own** account at that vendor. It is a
catalog connector like any other, with three differences `list_connectors` surfaces:

- **Custody.** The custody field says where the sign-in tokens of the apps live: "vendor vault" (the
  vendor's own store, in the region given) or "customer KMS" (the key stays in the organisation's
  cloud). The vendor is the organisation's processor, not PlatformSmith's sub-processor; an org admin
  accepts a versioned custody notice (the acknowledgement version on the row) when connecting the
  account. Connecting is done by a person in the UI (a server id, a user id and the vendor's API key)
  — **not a tool**.
- **Discovery.** The discovery flag is true: the entry curates no tools. `discover_connector_tools`
  reads them from the organisation's own aggregator server (the apps the admin configured there), one
  descriptor per tool named after the connector, the app and the action in that order (for Composio
  and GitHub: composio, github, create an issue), each with group = the app, **all switched off** until `review_connector_tool`. A
  tool without annotations counts as a write. Lists can run to hundreds of tools: enable per app, and
  expect approvals.
- **Account bindings.** A tool parameter that must equal the account's own vendor-side identity is
  pinned with `update_connector_tool` to the account's own vendor-side id (the bind source is the
  account's external id) or to a non-secret setting the admin filled at connect (the bind source is
  the account's config followed by the setting's name, such as Composio's user id); the gateway fills it per call from the account the session acts
  through.

A call to an app the user has not connected inside the aggregator is refused to the agent as
"connect required" with **no link**; the vendor's authorization link goes to the workspace's admins
through the notification plane. Per-call vendor pricing is attributable from the audit row (connector
+ account). A self-hosted aggregator reachable from the platform is simply a custom connector.

## Not a tool

**Connecting, re-authorizing or removing a connector account** needs a browser sign-in or a pasted
vendor key, so it is done by a person in the PlatformSmith UI under **Org → Connectors**. No tool
returns a token or key for any account — `list_connector_accounts` shows who an account is signed in
as and its status, never its credential.
