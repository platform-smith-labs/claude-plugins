# Connecting a client to ps-mcp

ps-mcp is a **remote Streamable-HTTP** MCP server, and there are now **two** ways to authenticate:

| | **Personal access token** | **Connected app (OAuth)** |
|---|---|---|
| How | you paste a `pat_…` bearer | the client runs an OAuth flow and you approve it on a consent screen |
| Good for | CLIs, CI, scripts, headless agents — anything that can set a header | hosted chat clients: Claude.ai, Claude Desktop and mobile, ChatGPT |
| Expiry | optional, up to 365 days | access token 1 hour, refreshed automatically; the connection itself lasts up to 365 days |
| Revoking | Account → Access tokens | Account → Connected apps |
| Availability | always | only where the deployment has enabled it |

**Neither replaces the other.** If your client can send a header, a PAT is simpler and is still the
right answer. OAuth exists because some clients *cannot*: ChatGPT supports OAuth only, and the
non-OAuth option in Claude.ai is a credential an org admin enters once and the whole organisation
shares — which would make every member act as whoever created it.

> **Which one will my client use?** It chooses. A client that finds no bearer header configured will
> try OAuth discovery; one you hand a PAT to will use the PAT. You do not pick a mode.

## Option A — connect as an app (OAuth)

Nothing to copy and paste. In the client, add PlatformSmith's MCP URL as a custom connector and
follow it:

1. The client discovers that the server is protected and finds the authorization server.
2. You sign in, if you are not already.
3. You **re-authenticate** — a password or a fresh provider round-trip. This is deliberate: approving
   an app mints a credential that acts as you for up to a year, and being signed in an hour ago is
   not proof you are the one approving it.
4. You see a consent screen naming the app, the host it will send you back to, and **which company**
   the connection acts in. You approve or cancel.
5. You get an email saying an app was connected. **If you did not connect it, that email is your
   signal** — disconnect it from Account → Connected apps.

Things worth knowing:

- **An app gets exactly what you have.** There are no read-only connections yet. Anything you can do
  in that company, the app can do.
- **Disconnecting takes effect on the app's next call.** There is no cache to wait out.
- **Only the apps the deployment allows** can be connected. That list is operator configuration, not
  something a user or an app can change.
- **If the app runs on your own computer** (Claude Code, for instance) the consent screen says so.
  Any local program can start that flow, so only continue if you just started it yourself.

If the client reports that it could not reach the server, the usual cause is that OAuth is not
enabled on that deployment — use a PAT instead.

## Option B — mint a PAT

PATs are minted through ps-api, **not** through this MCP server — issuing your own bearer token is a
privilege-escalation surface, so no tool here can do it.

**In the web UI**: avatar menu → **My access tokens** (or **Account → Access tokens**,
`/account/tokens`). Name it, pick an expiry, copy the value.

**Or over the API**:

```bash
# against whichever instance you are targeting
curl -s -X POST "$PS_API/api/v1/auth/login" -H 'Content-Type: application/json' \
  -d '{"email":"you@example.com","password":"…"}'        # -> {"token": "<jwt>"}

# expires_at is OPTIONAL and absolute (RFC3339). Omit it for a token that never expires.
curl -s -X POST "$PS_API/api/v1/user/pats" -H "Authorization: Bearer <jwt>" \
  -H 'Content-Type: application/json' \
  -d '{"name":"my-laptop","expires_at":"2026-12-13T00:00:00Z"}'   # -> {"token": "pat_…"}
```

The PAT value is shown **once**. Revoke with `DELETE /api/v1/user/pats/{user_pat_uuid}`.

### Expiry

`expires_at` must be in the future and at most **365 days** out; either bound returns 400. Omitting
it means the token never expires — still the default, and what every PAT minted before 2026-09-14
is.

An expired PAT is rejected exactly like a revoked one: the same uniform 401, with no way to tell
unknown from revoked from expired. Enforcement is per request (ps-mcp re-resolves on every call),
so an expiry takes effect on the very next call rather than after a token lifetime. A rejected
attempt does **not** advance `last_used_at`, so that column stays truthful after a token lapses.

`GET /api/v1/user/pats` returns a server-derived `status` on every row — `active`, `expired` or
`revoked` — alongside `expires_at`. Use `status`; do not recompute it from the timestamps, since it
is derived against the database clock that actually enforces expiry.

> Corrected 2026-09-14: an earlier note here claimed the listing carried no revoked marker. It
> does — `revoked`, `revoked_at` and now `status` — verified against a live response. The note was
> wrong, not the API.

## Once you have a credential — point a client at it

Every example below takes **the full MCP endpoint, including the `/mcp` path**. The path is not
cosmetic: an OAuth client compares it against the resource the server publishes, so a missing or
extra `/mcp` is a different resource and fails at authorization time.

```bash
PS_MCP=https://<your-platformsmith-host>/mcp
```

**Claude Code / Codex / Cursor** — direct bearer header:

```bash
claude mcp add --transport http ps-mcp "$PS_MCP" \
  --header "Authorization: Bearer $PS_MCP_PAT"
```

**Claude Desktop** (stdio-only) goes through the off-the-shelf bridge:

```bash
npx mcp-remote "$PS_MCP" --header "Authorization: Bearer $PS_MCP_PAT"
```

Verify the connection is real before trusting it — the handshake tells you the version you are
actually talking to:

```bash
curl -s -X POST "$PS_MCP" -H "Authorization: Bearer $PS_MCP_PAT" \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}'
```

## ⛔ Connecting an AI client to a tenant with real content is a data-custody decision

This is not a warning about breaking something. It is about what flows **out**.

A credential here authorises the full tool surface **as you**, and that surface includes
`get_session_events`, `list_session_artifacts`, `get_artifact` and the work-item and conversation
reads. Those return real content — session transcripts, prompts, agent output, source code, user
names and email addresses. Whatever your assistant reads is in a third-party model's context from
that moment on, and you cannot take it back.

So, before connecting:

- **Decide it deliberately**, the way you would any other export of that data. Your organisation may
  already have a rule that answers this.
- **Do not** wire a production credential into a shared or default client config, where somebody
  else's session inherits it.
- **Prefer the envelope over the payload** when debugging. `event_type`, `severity`, `phase`,
  `created_at` and the sequence number usually tell you where a run stopped without reading what it
  said.
- **Prefer a workspace whose data you are free to expose** while you are still learning the tools.

