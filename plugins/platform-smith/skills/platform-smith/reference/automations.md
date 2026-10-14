# Automations — make something happen on its own

An **automation** connects a **source** to a **target**:

| Source (`source.kind`) | Fires on |
|---|---|
| `github_repo` | pull-request, push, PR-comment and review events on a project's linked GitHub repository |
| `bitbucket_repo` | the same events from a Bitbucket repository |
| `jira` | comments on issues in one Jira project |
| `linear` | comments on issues in one Linear team |
| `github` | comments on issues in one GitHub repository (GitHub *issues*, as a tracker) |
| `slack` | alerts posted in one Slack channel (`mode: alert`), or @mentions in it (`mode: mentions`) |
| `schedule` | a 5-field cron expression in an IANA timezone |
| `webhook` | a signed HTTP POST from any system that can send one |

| Target (`target.target_type`) | Starts |
|---|---|
| `session` | a coding-agent session, prompted from `prompt_template` (which may use `{{event.*}}`) |
| `workflow` | a run of a published workflow definition |
| `playbook` | a playbook run |
| `none` | **nothing** — record only: each event creates or links a work item (see below) |

Workflow and playbook targets receive `input_defaults` plus the normalized `event`
(repository, pull request, push, comment, review, sender, command). Text that other people
wrote — pull-request titles and bodies, comments, labels, command arguments — reaches a
session's agent marked as untrusted input.

`list_automations` shows every automation in a workspace, whatever its source — including one
you did not create: the **built-in Slack conversation automation** (`origin: built_in`). That one
is what lets people in the workspace start work with **@mentions and `/smith`** in Slack. It can
be switched on and off (`enable_automation` / `disable_automation`) but never edited or deleted
(`422 built_in_automation`). Switched off, people who mention the app are told mentions and
`/smith` are off in this workspace.

## The one thing to get right: enabling is a decision

An **enabled** automation starts its target **unattended** — nobody watches the run start —
every time its source fires. A coding agent may push to a repository with nobody watching.

So the pattern is always:

1. `create_automation` with `"enabled": false`.
2. Show the person what it will do: the source, its filters or command, the target, and
   **who it acts as** (next section).
3. Only when they agree, `enable_automation`. The platform records who switched it on and when.

`create_automation` therefore always creates switched off — `"enabled": true` is refused, and
an omitted flag is sent as `false`. `enable_automation`, `resume_automation`,
`run_automation_now`, `update_automation` and `delete_automation` are marked destructive for
this reason. `disable_automation` is the safe way to stop one: nothing
new starts, runs already in flight continue, and its history is kept.

**Credentials warn, they never refuse.** If the service principal has no shared coding-agent
credential, the automation is still saved and the response says so. Tell the person: every run
that acts as the service principal will fail, and the delivery log will show `failed` /
`no_credential`, until a shared credential exists for the workspace.

## Require approval: hold each run until someone says yes

`policies.require_approval` makes an enabled automation **ask before every run** instead of starting
it unattended:

```json
"policies": {"require_approval": {"timeout": "24h"}}
```

- Each matching event creates a run with status **`held`** and raises a **workspace notification**
  ("Approve run of …?") naming the event, who caused it and when the hold expires — never the pull
  request's text or the prompt.
- **Approve** starts it, acting as the identity decided when it was held (the person who caused it,
  or the service principal) — never as the approver, who is recorded as `approved_by`.
  **Decline** drops it (`declined`). Nobody answering before `timeout` **expires** it (`expired`);
  an expired run never starts.
- Answer it with `approve_automation_run` / `decline_automation_run`, or with
  `answer_notification` on the run's `approval_notification_uuid` using `choice: "approve"` or
  `"decline"`. They are the **same decision**: the first answer wins and a second one gets
  `409 already_settled`. `409 not_held` means the run is no longer waiting; `403` means the caller
  cannot answer this workspace's notifications.
- **Anyone who can answer the workspace's notifications may approve — including whoever (or
  whichever agent) raised the event.** There is no separate approver list. `can_approve` on the run
  says whether you can. Still ask the person before approving: an approved run starts unattended.
- `timeout` is a duration from `1h` to `168h` (7 days); `null` switches approval off. Caps, the
  minimum interval and concurrency are checked again at approval, so an approved run can still be
  dropped (`cap_reached`, `min_interval`, `concurrency_skip`). Record-only automations never hold.
- `run_automation_now` on such an automation answers `held: true` with the held run's `run_uuid`.

## Who a run acts as

A run acts **as a person** only when a person caused it, and only if the platform can confirm
who they are:

- **A person who explicitly asked** — typed a command in a comment, @mentioned the app in Slack,
  or used `/smith` — acts as themselves.
- **A pull request's author** acts as themselves for that pull request's events on a
  repository automation that starts a session.

…provided that person's provider account (GitHub, Bitbucket, Jira, Linear or Slack user) is
**linked** to a PlatformSmith user who is allowed to start the target in this workspace.
Otherwise the run acts as the **workspace's service principal**, and the run records why.

Everything with no person behind it — schedules, webhooks, Slack alerts, pushes, and
`run_automation_now` — always acts as the service principal. (Whoever pressed Run now is
recorded, but the run does not act as them.)

`get_automation` shows this as a read-only **`identity`** object, which the platform derives:

| `identity` field | Meaning |
|---|---|
| `acts_as: service_principal` | every run acts as the workspace's service principal |
| `acts_as: invoker_when_linked` | a run acts as the person who caused it, when linked and allowed |
| `invoker` | which person that is: `pr_author`, `commenter` or `mentioner` |
| `skip_unlinked_authors` | `true`: a pull-request event from an unlinked author is **dropped** instead of running as the service principal |

Do not send `identity` back in an update — it is not configuration. To change who runs act as,
change the source or target; to stop unlinked authors' pull requests running at all, set
`policies.skip_unlinked_authors: true` (repository sources).

Slack @mentions and `/smith` are stricter: they **never** fall back to the service principal.
If the sender cannot act as themselves, nothing starts and the delivery log says why.

When a person acts as themselves, their **own** coding-agent credential is used only if they
have allowed it for automations; otherwise the workspace's shared credential is.

## Commands: `<prefix> <verb> [args]`

Repository and tracker automations can answer a **command** typed in a comment:

```
/ps review security
```

- `/ps` is the **command prefix**. It is set for the organisation and a workspace may override
  it — read the one that applies with `get_workspace_command_prefix` (`effective`) before you tell
  anyone what to type. The prefix is case-sensitive.
- `review` is the automation's **`source.command_verb`** (case-insensitive).
- `security` — everything after the verb — reaches the target as `event.command.args`
  (`{{event.command.args}}` in a prompt template), marked as untrusted.

A comment that starts with the prefix but names a verb **no** automation on that repository or
tracker container answers is logged as `unknown_command`. On a repository automation without a
`command_verb`, ordinary comments still match, but another automation's command does not.

`create_automation` and `update_automation` may answer with **`warnings`** — the save
succeeded, but the person should know:

| Warning `code` | Meaning |
|---|---|
| `duplicate_verb` | another automation on the same repository or container already answers this verb |
| `prefix_mismatch` | automations on the same repository or container would answer different command prefixes |

Pass the `message` on. (`filters.comment_command`, the older way to match a comment, is gone:
use `command_verb`.)

Labels and assignments on tracker issues **do not start automations**: such an event only captures the
ticket as a work item, and no delivery is logged. Use a command.

## Work items: attach each run to the work it belongs to

`work_item` decides how an automation's events attach to **work items**:

```json
"work_item": {"mode": "link", "item_type_name": "Bug"}
```

- `mode: "link"` creates or links **one work item per subject** and attributes the run to it: the
  pull request, the issue, the Slack alert, the schedule fire, or for a webhook the value of
  `key`. `mode: "none"` creates nothing.
- `item_type_name` picks the type a new item gets — `list_workspace_item_types` lists them.
- `key` — **schedule**: `per_fire` (one item per fire date) or `fixed` (one item for the
  automation); **webhook**: an expression over the event whose value identifies the item, such as
  an incident id, so repeat deliveries about the same thing share one item.
- Tracker automations always link. A work item that cannot be linked never stops a run.

`get_automation` always returns `work_item` (each source kind has a default).

### Record only

With `"target": {"target_type": "none"}` an automation **starts nothing**: each event just
creates or links its work item. It needs `work_item.mode: "link"`, takes no agent, environment,
prompt or inputs, and may name a `project_uuid` for the item. Its deliveries show `recorded` /
`record_only`. Use it when the person wants every alert or ticket tracked, but not acted on yet.

## Recipe: review every pull request

1. Find the project and confirm its repository is linked: `list_projects`, then
   `get_project_git_link`. A repository source watches the project's linked repository.
2. Pick the target. Either a playbook (`list_workspace_playbooks`) or a coding-agent session —
   for a session, choose an agent definition whose `harness_credential_configured` is `true`
   and an environment (`list_environments`).
3. Create it switched off:

   ```json
   {
     "name": "Review PRs",
     "enabled": false,
     "source": {
       "kind": "github_repo",
       "project_uuid": "<project>",
       "events": ["pull_request.opened", "pull_request.synchronize"],
       "filters": {"base_branches": ["main"], "include_drafts": false}
     },
     "target": {"target_type": "playbook", "target_playbook_uuid": "<playbook>"},
     "work_item": {"mode": "link"}
   }
   ```

   For a Bitbucket repository use `"kind": "bitbucket_repo"` with the same fields.
4. Read it back with `get_automation` — including `identity` — walk the person through it, then
   `enable_automation`.

Defaults that matter here: a new push to the same pull request **supersedes** the run already
in flight (concurrency key `pr`), events caused by the platform's own App are ignored (loop
guard — an agent's own push does not re-trigger review), and pull requests from forks start
nothing unless the fork policy allows them.

`list_automation_suggestions` lists projects whose linked repository has no automation yet,
each with a suggested configuration. Accepting one is `create_automation` with
`"origin": "suggested_on_connect"` and `"enabled": false` — a suggestion never switches itself on.
Suggestions someone dismissed for the workspace are left out (`dismissed_count` says how many);
`include_dismissed=true` lists them too. Dismissing is done on the Automations page in the app.

## Recipe: a command people type on a pull request or ticket

1. `get_workspace_command_prefix` — note `effective` (for example `/ps`).
2. Create it switched off, with the verb people will type:

   ```json
   {
     "name": "Fix on request",
     "enabled": false,
     "source": {
       "kind": "jira",
       "account_uuid": "<jira site account>",
       "conversation_ref": "<the Jira project URL>",
       "command_verb": "fix"
     },
     "target": {"target_type": "session", "agent_definition_uuid": "<definition>",
                "environment_uuid": "<environment>", "project_uuid": "<project>"},
     "prompt_template": "Fix {{event.command.args}}"
   }
   ```

   The account comes from `list_integration_connections` or `list_workspace_integrations`. For
   Linear, `conversation_ref` is the team key; for GitHub issues (`"kind": "github"`), the
   `owner/repo`. For a pull-request command, use a repository source with
   `"events": ["pull_request_comment.created"]` and the same `command_verb`. The account and
   container cannot be changed later — create a new automation instead.
   A tracker source may also set `inbound_mode` (`capture`, the default, records the ticket's later
   comments on the work item; `live` also passes them to a running session), `work_item.item_type_name`
   for the type a captured ticket becomes, and `report_back` (below).
3. Check the response for `warnings`, and for a tracker source check `webhook_registered`: the
   platform registers the provider webhook on create, and without it the ticket's comments never
   arrive. If it is `false`, `webhook_registration_cause` says why (`auth_rejected`, `not_connected`,
   `container_not_found`, …); once the cause is fixed, `register_automation_webhook` retries it in
   place. Tell the person what to type (`/ps fix …`), and explain who it will act as: the commenter,
   if their account is linked.
4. `enable_automation` once they agree.

## Report back: what a run writes to the source

`report_back` decides what the platform writes back where the event came from:

| Source | `report_back` | Meaning |
|---|---|---|
| `jira`, `linear`, `github` | `beacon_mode`: `off` · `suggest` (default) · `auto` | the work item's status beacon on the ticket |
| | `findings`: `off` (default) · `on` | the run's findings posted as a ticket comment |
| `github_repo` | `checks`: `true` · `false` (default) | each run reported as a check on the pull request |
| | `pr_comment`: `true` · `false` (default) | the summary the agent reported at the end of the run, posted as one pull-request comment |
| `slack` | — | replies in the alert's thread on its own |

`pr_comment` posts only what the agent deliberately reported as its outcome (its `verdict_summary`,
below) — no summary, no comment, and never the transcript. It is refused (422) on any source but
`github_repo`. With `checks` on, the check's conclusion follows the agent's verdict: `pass` →
success, `fail` → failure, `needs_attention` → action required.

On an update, a tracker's `beacon_mode`, `findings` and `inbound_mode` keep their stored values when
left out, so a read sent back never switches them off by accident. (`writeback_mode`, the older
GitHub field, is gone and refused: use `report_back.checks`.)

## Recipe: act on alerts in a Slack channel

1. Find the Slack account (`list_workspace_integrations`) and the channel id.
2. Create it switched off:

   ```json
   {
     "name": "Triage alerts",
     "enabled": false,
     "source": {
       "kind": "slack",
       "account_uuid": "<slack account>",
       "conversation_ref": "C0123",
       "mode": "alert",
       "who_may_trigger": [],
       "routing_rules": [
         {"action": "drop", "label_conditions": [{"operator": "contains", "value": "[test]"}]},
         {"action": "route", "label_conditions": [{"operator": "contains", "value": "payments"}],
          "target": {"project_uuid": "<payments project>"}}
       ],
       "on_no_match": "default_target"
     },
     "target": {"target_type": "session", "agent_definition_uuid": "<definition>",
                "environment_uuid": "<environment>", "project_uuid": "<project>"},
     "work_item": {"mode": "link"}
   }
   ```

   Rules are checked in order and the first match wins: `drop` ignores the alert, `route` sends it
   to its own target (a `null` field inherits the automation's). `on_no_match` decides what an
   alert no rule matched does. A rule target must stay inside `allowed_agent_definitions` /
   `allowed_environments`. `who_may_trigger` limits whose posts count (empty: anyone). Slack takes
   no `prompt_template` and no `report_back`.
3. With `"mode": "mentions"` the channel answers @mentions instead, each acting as the person who
   mentioned (never falling back to the service principal). Alerts posted there are then ignored.
4. `list_automation_deliveries` names the rule that decided each alert (`rule {priority, action}`).

## Recipe: run something on a schedule

1. Check the expression with `preview_cron` (`cron` + `timezone`) and show the person the
   next fire times. It is the same parser the scheduler uses, with daylight-saving changes
   applied — `0 9 * * 1-5` in `Europe/London` fires at 09:00 London time on weekdays all year.
2. Create it switched off:

   ```json
   {
     "name": "Weekday triage",
     "enabled": false,
     "source": {"kind": "schedule", "cron": "0 9 * * 1-5", "timezone": "Europe/London",
                "overlap": "skip", "catch_up": "once"},
     "target": {"target_type": "workflow", "target_workflow_definition_uuid": "<definition>"}
   }
   ```

   `overlap` decides what a fire does while the previous run is still live (`skip` by default,
   `queue` one, or `allow`). `catch_up` decides what happens after a missed fire (`once` fires one
   catch-up run, `skip` fires none).
3. `run_automation_now` fires it once immediately to try it. It answers `{run_uuid}` (plus
   `work_item_uuid` when it links one) — or, for a record-only automation, `{recorded: true,
   work_item_uuid}` with no run, because nothing runs. A record-only **webhook** automation cannot
   be run now (422): there is no event to key its work item on. Run now ignores the overlap policy
   but respects the caps.
4. `enable_automation` once the person is happy.

After repeated consecutive failures (3 by default) the platform **auto-pauses** the automation
and notifies its owner; `get_automation` shows the pause. Find the cause first, then
`resume_automation`.

## Webhooks and their secrets

A `webhook` automation verifies each delivery with a signing secret (`github`, `stripe`,
`standard_webhooks`, `hmac` or the weaker `token` verifier). You can create one here only when
the person already holds the secret and pastes it (`"secret_mode": "paste"`). A **generated**
secret is shown exactly once, and this server never returns a secret value — so creating a
webhook with a generated secret, and rotating a secret, are done on the **Automations page in
the PlatformSmith app**. Ask the person to do it there. No tool ever returns a stored secret.

## "Why didn't it fire?" — read the delivery log

`list_automation_deliveries` shows the latest 50 deliveries for one automation, newest first.
Each row is **envelope only** — the event name, the outcome, the reason, the time, the run it
started and the work item it linked, plus `event_name` (such as `pull_request.synchronize`),
`event_ref` (a pull request `#12` or a branch), a short `head_sha` and `sender_login`. It never
contains the event payload or anyone's text.

Events this automation does not listen for (`no_event_match`) are **hidden by default**:
`ignored_count` says how many arrived in the last 14 days, and `show_ignored=true` lists them.

| Outcome / reason | Meaning | What to do |
|---|---|---|
| `started` | a run began (reason `approved` when a held run was approved) | follow it with `get_automation_run` |
| `held` / `awaiting_approval` | `require_approval` is holding the run | approve or decline it (above); a second row records how it settled |
| `dropped` / `approval_declined` | someone declined the held run | expected |
| `dropped` / `approval_timed_out` | nobody answered before the hold expired | expected — lengthen `timeout` if it keeps happening |
| `recorded` / `record_only` | a record-only automation linked its work item; nothing ran | expected |
| `dropped` / `disabled` | the automation is off | `enable_automation`, with the person's agreement |
| `dropped` / `built_in_off` | @mentions and `/smith` are switched off in this workspace | switch the built-in automation on, if the person wants them |
| `dropped` / `paused` | auto-paused after failures | fix the cause, then `resume_automation` |
| `dropped` / `no_event_match` | the event is not in `source.events` | add the event |
| `dropped` / `filtered` | a filter excluded it (base branch, labels, paths, authors, draft), or the comment was not this automation's command | compare the event with `source.filters` and `command_verb` |
| `dropped` / `unknown_command` | the comment used the prefix with a verb no automation here answers | check the verb, and the prefix with `get_workspace_command_prefix` |
| `dropped` / `author_not_linked` | `skip_unlinked_authors` is on and the pull request's author has no linked PlatformSmith user | the author links their account in the app — or turn the policy off |
| `dropped` / `person_not_linked`, `person_not_member`, `person_unchecked` | a Slack @mention or `/smith` whose sender could not act as themselves (see the run table below) | the person links their account, or is given access to the target |
| `dropped` / `loop_guard` | the platform's own App caused the event | expected — this is what stops loops |
| `dropped` / `fork_policy` | a pull request from a fork | change the fork policy only if the person wants forks reviewed |
| `dropped` / `cap_reached`, `min_interval` | a rate cap or minimum interval | expected under load; adjust the policy if not |
| `dropped` / `overlap_skipped`, `concurrency_skip` | a previous run was still live | expected with `skip` |
| `dropped` / `missed_fire_skipped` | a schedule fire was missed and `catch_up` is `skip` | expected |
| `dropped` / `cel_filter` | a webhook filter expression said no | check the filter |
| `suppressed` / `deduplicated` | a repeat of a Slack alert already handled | expected |
| `failed` / `no_credential` | no usable coding-agent credential for whoever the run acts as | ask an admin to add a shared one |
| `failed` / `person_credential_not_granted` | the run acted as a person whose own coding-agent credential is not allowed for automations | that person allows it in the app |
| `failed` / `target_gone` | the target was deleted | point the automation at a live target |
| `failed` / `principal_unavailable` | the workspace's service principal could not be resolved | ask an admin to check the workspace |
| `failed` / `work_item_failed` | a record-only delivery could not link its work item | check `work_item` (type name, key) |
| `failed` / `dispatch_failed` | the target could not be started | read the run and the target's own status |

If an event **never appears** in the log, it never reached this automation. For a repository
source, check that the project is still linked to that repository, that the platform's GitHub
App is installed on it (for Bitbucket: that the repository still carries the hook the platform
registered when the automation was created), and that the integration account has **Receives
events** on (`get_integration_account`). A webhook delivery with a bad signature or an unknown
address is rejected before it is logged.

## What it started, and who it ran as

`list_automation_runs` shows what the automation started or is holding: each run's status
(`live`, `succeeded`, `failed`, `cancelled`, `superseded`, `unknown`, and with approval `held`,
`declined`, `expired`), its `event_name`, its `target_ref` — open it with `get_session`,
`get_workflow_execution` or `get_playbook_run` — and its `work_item` (open it with
`get_work_item`). `superseded` means a newer push to the same pull request replaced it.
`get_automation_run` reads one run by `run_uuid`.

A held run also carries `hold_expires_at`, `approval_notification_uuid`, `requested_by` (the person
who caused it, or `null`) and `can_approve`; once settled, `approved_by` and `decided_at`.

**The agent's verdict.** A coding agent started by an automation can report its own outcome —
`verdict` `pass`, `fail` or `needs_attention` (or `null` when it reported none), with a
`verdict_summary` it deliberately wrote and `verdict_at`. A `fail` verdict makes the run `failed`.
`pr_comment_url` links the pull-request comment posted from that summary when
`report_back.pr_comment` is on.

Each run also says **who it ran as**: `identity` (`person` or `service_principal`), `acted_as`
(the user it acted as), and — when a person asked but the run fell back to the service
principal — an `identity_note`:

| `identity_note` | Meaning | What to do |
|---|---|---|
| `person_not_linked` | the person's provider account is not linked to a PlatformSmith user | they link it in the app (`list_integration_account_people` shows who is linked) |
| `person_not_member` | linked, but not allowed to start this target in this workspace | an admin gives them access, if they should have it |
| `person_unchecked` | the platform could not check their permission at that moment, so the run acted as the service principal | usually transient; check the next run |
| `person_target_unsupported` | this target does not act as a person for that kind of event | expected |
