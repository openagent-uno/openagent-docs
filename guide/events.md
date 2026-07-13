# Events (webhook triggers)

An **Event** is a first-class inbound trigger. Where a [scheduled task](./scheduler.md)
fires *when the clock says so* and a workflow runs
*when you say so, in a fixed shape*, an event fires **when the world says so** —
an external service (or a peer agent) calls in and the agent does bound work.

Every event has:

- a **name** and a URL **slug** (`POST /hooks/{slug}` on the [webhook listener](./channels.md#webhook-inbound)),
- a webhook **type** preset that decides how a delivery is authenticated,
- a per-event **secret** (shown once at creation; stored encrypted at rest),
- an optional **input schema** documenting the payload you expect,
- an **action**: run a workflow, fire a scheduled task, or start a chat prompt.

## The three actions

| Action | What fires | Payload |
|---|---|---|
| **Workflow** | an existing workflow run | delivered as the workflow `inputs` — reach it with `{{inputs.<field>}}` in any block |
| **Scheduled task** | the task's prompt, out of band from its cron | appended to the prompt as an injection-guarded block |
| **Chat prompt** | a fresh, durable child session | rendered into the prompt via `{{payload.<field>}}` |

The chat-prompt action runs as a real session: it appears in the sidebar
history like any other execution, streams live, and can be opened and continued.
Every delivery — whatever the action — is recorded in a durable delivery history
and surfaces in the app's Recent feed under its own **Events** filter.

## Authentication

Two layers, checked in order:

1. **Secret.** Every event carries a per-event secret. A caller presents it in
   `X-OpenAgent-Event-Secret` (or `Authorization: Bearer …`). This is the
   baseline for `type: generic`.
2. **Signature.** Provider presets add an HMAC check over the *raw* body, which
   a real sender satisfies automatically:
   - `github` — verifies `X-Hub-Signature-256`, dedupes on `X-GitHub-Delivery`.
   - `stripe` — verifies the timestamped `Stripe-Signature` (±5 min window).
   - `slack` — verifies `X-Slack-Signature` (±5 min window).
   - `generic-hmac` — a portable `X-Signature-256: sha256=<hmac>` scheme.

The secret is stored **encrypted at rest** (not a one-way hash), because HMAC
verification needs the key in clear; only a 4-character hint is kept in the open
for the UI. Guardrails per event: a payload size cap (413 on overflow), a
per-minute rate limit (429), and idempotent de-duplication on the provider's
delivery id (a redelivery is acknowledged without re-running).

**Never** appears in a read: `GET /api/events` returns only the hint. The clear
secret is returned once, inline, on create and on rotate. Rotating a secret
invalidates the old one immediately.

## Triggering from the Iroh network

External HTTP is one doorway; the mesh is the other. A member device or an
invited peer agent can fire an event over Iroh with
`POST /api/events/{id}/trigger` (body `{"payload": {…}}`), authenticated by its
device certificate — no shared secret needed, consistent with OpenAgent's
peer-to-peer model. The delivery, history, and produced run are identical to a
webhook-triggered one.

## Managing events

- **App** — the **Events** screen (under Workflows in the sidebar): create,
  edit, toggle, rotate the secret, and send a test delivery. Configure the
  listener under Settings → Channels → Webhook.
- **CLI** — the `/events` menu: add, edit, fire, rotate, view history, delete.
- **Agent** — the `events-manager` MCP: `create_event`, `update_event`,
  `delete_event`, `list_events`, `enable_event` / `disable_event`,
  `rotate_event_secret`, `trigger_event`, `list_event_deliveries`. So the agent
  can wire up "when GitHub pushes, run my review workflow" on its own.

## Example

```bash
# Create in the app (or ask the agent): type=github, action=workflow,
# target=code-review. Copy the secret shown once. Then in GitHub:
#   Settings → Webhooks → Payload URL = https://hooks.example.com/hooks/github-push
#                         Secret       = whsec_…
#                         Content type = application/json

# GitHub delivers on every push; OpenAgent verifies the signature, dedupes on
# the delivery id, and runs the code-review workflow with the push payload as
# inputs. The run shows up in the sidebar Recent feed under Events.
```
