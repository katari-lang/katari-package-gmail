# gmail — Gmail tools for Katari

A single module, `gmail`: four tools the model can call — `list_recent`, `read_body`, `fetch_attachment`,
`send` — plus label editing (`modify_labels`) and a notification watcher (`watch`) over Gmail's REST API.
Pure Katari: every API call is `http.fetch` with the request built and the reply parsed as `json`. No FFI
sidecar, and no OAuth plumbing in the program — authentication is the runtime's credentials core, reached
through the stdlib's `oauth.token`.

- `gmail.list_recent(query, max_results ?= 10)` — the messages matching a Gmail search, newest first, each
  trimmed to `id` / `thread_id` / `message_id` / `sender` / `subject` / `snippet` / `date` / `attachments`.
  `query` is the search-box language (`"is:unread"`, `"from:alice@example.com newer_than:2d"`).
- `gmail.read_body(id)` — one message's body as a `message_body(content_type, text)`.
- `gmail.fetch_attachment(id, attachment_id, content_type ?= null)` — one attachment as a `file`, fetched
  by the message's `id` and the `attachment_id` of an entry in its `attachments`.
- `gmail.send(to, subject, body, thread_id ?= null, in_reply_to ?= null)` — send plain-text mail,
  returning the sent message's id; pass `thread_id` + `in_reply_to` to reply inside a conversation.
- `gmail.modify_labels(id, add ?= [], remove ?= [])` — add and remove label ids. Removing `"UNREAD"` marks
  a message read. There is no delete, at any privilege.
- `gmail.watch(query, poll_interval_milliseconds, cursor_at, deliver_to)` — a daemon that delivers each
  arriving message matching `query` to `deliver_to(value = …)`, oldest first. Never resolves; composes
  under `parallel [ … ]`.
- `gmail.provider(source = ...)` — provides the capability the tools require (`credential`) for the extent
  of a continuation, by resolving a `credentials.source` through the runtime on every ask. It holds no
  secret and keeps no cache: the runtime owns the token material and its refresh, and the resolved token is
  a `string of private` that flows only to the `Authorization: Bearer` header.

The same stored Google credential serves `google_calendar`; give it a Gmail scope and both read through one
login. They also share their plumbing — the `Bearer` header, the 401/403 classification and the
authenticated call — in `google_common`, which arrives as their transitive dependency.

## The message

`read_body` returns data rather than a formatted string, so the caller dispatches on the type it was handed:

| `content_type` | `text` is |
| --- | --- |
| `"text/plain"` | the message's plain-text body, decoded |
| `"text/html"` | the message's markup, tags and all — it had no plain-text part |
| `null` | Gmail's short snippet preview — the message had neither part |

Render the markup, strip it, or hand it to a model, which reads HTML fine. Nothing is truncated.

Every message also lists what it carries — `attachment(filename, content_type, size, attachment_id)`, one
per part whose bytes Gmail holds behind an attachment id, inline images included — so a mail can be judged
before any bytes move. `fetch_attachment` takes that `attachment_id` with the message's own `id` and
returns a `file`, a handle whose bytes stay runtime-side: the payload never enters the conversation, a
durable call record or a trace, and the handle passes straight to any tool that takes a file. It records
the content type the mail declared, which is what lets an image be recognised downstream; pass that type in
from the listing, since Gmail regenerates attachment ids on every message read.

`send` writes an RFC 5322 message and posts it as `messages.send`, plain text only. Compose the `subject`
yourself — for a reply, prefix `"Re: "` — and pass both threading values from the message you are
answering: `thread_id` files the reply in that Gmail conversation, and `in_reply_to` (the parent's
`message_id`, its RFC822 `Message-ID`, not its `id`) writes the `In-Reply-To` / `References` headers so
every mail client threads it.

## Failures

Three meanings, decided at three boundaries:

- A credential that needs a human — never authorized, or its refresh is dead — pauses the run on a
  `prelude.oauth.authorize` escalation, which the admin console and `katari answer` render as an
  authorization request; completing the browser flow resumes the run where it stopped.
- `oauth.server_error` (stdlib) — the token could not be resolved for a transient reason. Thrown at the
  provider.
- `http.api_failure = http.auth_error | http.api_error` (stdlib) — the Gmail API's own failures, classified
  by `http.classify_status` where `google_common.call` reads a response, both carrying `status`, `context`
  and `message`. `http.auth_error` is a 401/403: replaying the call resolves the token afresh, the runtime
  refreshes the credential once its stored lifetime passes, and one it cannot refresh parks the run as a
  re-authorization prompt. `http.api_error` is any other non-2xx; read `status`, since 408 / 429 / 5xx are
  worth a backoff.

The pair is the stdlib's, so one `supervise` converter covers `google_calendar` and every other
authenticated REST package beside this one. `fetch_attachment` adds `files.malformed_base64`.

## Secrets / env

None in the program: the runtime hosts the OAuth flow — browser consent, redirect callback, token exchange,
storage and refresh — and the program names the credential.

1. In [console.cloud.google.com](https://console.cloud.google.com), create or select a project and enable
   the **Gmail API**. Configure the OAuth consent screen; a Testing app's refresh tokens expire after 7
   days, so publish the app (or use Internal on a Workspace domain) for a daemon that keeps running.
2. Create an **OAuth client ID** of type Web application, with the runtime's callback as an authorized
   redirect URI: `<public-url>/oauth/callback` (`KATARI_PUBLIC_URL`; locally
   `http://localhost:3000/oauth/callback`). Note the client id and secret.
3. In the admin console, open the project's **Credentials** page and press **Register client**:

| Field | Value |
| --- | --- |
| Name | `google` (whatever name the program passes to `credentials.oauth`) |
| Issuer | `https://accounts.google.com` |
| Authorization endpoint | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token endpoint | `https://oauth2.googleapis.com/token` |
| Client ID / Client secret | from the console (the secret is write-only) |
| Scopes | `https://www.googleapis.com/auth/gmail.modify` |
| Extra authorize parameters | `access_type=offline prompt=consent` |

`gmail.modify` is Google's read/write scope short of permanent deletion and authorizes every call this
package makes; an app that only reads can register `gmail.readonly` and not call `send` / `modify_labels`.
`access_type=offline` is what makes Google issue a refresh token, and `prompt=consent` makes it re-issue one
on a later re-authorization. Sharing the credential with `google_calendar` means one client with both scopes
registered; adding a scope requires logging in again.

Press **Log in** on the client's row to store the credential, or let the first `credential` ask of a run
pause on its authorization escalation and log in from there. For several Google accounts, register the same
client under several names and pick one per scope.

## Usage

```katari
import gmail

agent recent(query: string) -> array[gmail.message] with io | prelude.throw[http.api_failure | oauth.server_error | env.missing_secret | http.fetch_error | json.parse_error] {
  use gmail.provider(source = credentials.oauth(name = "google"))
  gmail.list_recent(query = query, max_results = 5)
}
```

Hand the five tools to an AI loop's tool list to let the model read and answer mail on its own; each carries
a model-facing `@doc`. Reading, labelling and sending differ in consequence, so an app that gates anything
gates the send.

## The watch

`gmail.watch` is `poll.subscribe` — the stdlib's durable-subscription skeleton — under a Gmail adapter, so
it speaks that module's guarantees. What it delivers is an arrival: the subscription primes a cursor when it
starts and each tick searches `<query> after:<cursor>`, so the mailbox as it stood at the start sits below
the cursor, as does mail that comes to match `query` later; `list_recent` answers "what matches this
search". The cursor advances by the delivered messages' own `internalDate`, Gmail's receive clock, never by
the poller's, and its only companion memory is the ids inside the 1000 ms overlap each fetch re-lists.

Delivery is at-least-once. The commit lands only after a delivery returns, so a crash or a throw in between
re-delivers exactly that message on the next tick; exactly-once is built at the destination, by making the
delivery idempotent on the message's `id`.

`cursor_at` decides what a restart does, and has no default. `poll.resume(key = "…")` keeps the cursor in
the store, so a runtime restart, a re-forked fiber and a `replay` re-run all resume from it and the mail
that arrived while the watch was down is delivered, oldest first, on the tick after it comes back — give
each watch a key of its own. `poll.fresh()` keeps it in the run alone, re-priming on every fresh activation.

Within one activation a tick drains every page of its listing, so a watch that resumes after a long gap
catches up on the whole backlog, oldest first. The first poll is one `poll_interval_milliseconds` in.

```katari
import gmail

@"Mark every arriving promotion read as it lands."
agent file_arrivals() -> never with io | store.get | store.set | prelude.throw[http.api_failure | oauth.server_error | env.missing_secret | http.fetch_error | json.parse_error] {
  use gmail.provider(source = credentials.oauth(name = "google"))
  agent mark_read(value: gmail.message) -> null with gmail.credential | io | prelude.throw[http.api_failure | http.fetch_error | json.parse_error] {
    gmail.modify_labels(id = value.id, remove = ["UNREAD"])
  }
  gmail.watch(
    query = "category:promotions",
    poll_interval_milliseconds = 300000.0,
    cursor_at = poll.resume(key = "promotions-watch"),
    deliver_to = mark_read,
  )
}
```

`deliver_to`'s effects flow out to the caller's handlers unchanged, so a delivery may itself read the body,
fetch an attachment, reply or label. A poll failure propagates and kills the watch; resilience is composed
around it with a `prelude.supervise` provider and a converter that replays the transient failures and
`http.auth_error` alike. With `cursor_at = poll.resume(...)` the backoff costs only latency, since the
cursor outlives the failed activation.

## License

MIT.
