# gmail — Gmail tools for Katari

A single module, `gmail`: four tools the model can call — `list_recent`, `read_body`,
`fetch_attachment`, `send` — plus label editing (`modify_labels`) and a notification watcher (`watch`)
over Gmail's REST API. Pure Katari: every API call is `http.fetch` with the request built and the reply
parsed as `json`. No FFI sidecar — and no OAuth plumbing in the program: authentication is the runtime's
credentials core, reached through the stdlib's `oauth.token`.

- `gmail.list_recent(query, max_results ?= 10)` — the messages matching a Gmail search, newest first,
  each trimmed to `id` / `thread_id` / `message_id` / `sender` / `subject` / `snippet` / `date` /
  `attachments`. `query` is the **search-box language** (`"is:unread"`,
  `"from:alice@example.com newer_than:2d"`, `"subject:invoice has:attachment"`).
- `gmail.read_body(id)` — one message's body as a `message_body(content_type, text)`: the `text/plain`
  part when there is one, else the `text/html` part's markup, else Gmail's snippet — and the type says
  which (see [The body](#the-body)).
- `gmail.fetch_attachment(id, attachment_id, content_type ?= null)` — one attachment as a **`file`**, fetched by the message's
  `id` and the `attachment_id` of an entry in its `attachments` (see
  [Attachments](#attachments)).
- `gmail.send(to, subject, body, thread_id ?= null, in_reply_to ?= null)` — send plain-text mail,
  returning the sent message's id; pass `thread_id` + `in_reply_to` to **reply inside a conversation**.
- `gmail.modify_labels(id, add ?= [], remove ?= [])` — add and remove label ids. Removing `"UNREAD"` is
  how a message is marked read; there is no delete, at any privilege.
- `gmail.watch(query, poll_interval_milliseconds, cursor_at, deliver_to)` — a daemon that delivers each
  **arriving** message matching `query` to `deliver_to(value = …)`, oldest first. It is `poll.subscribe`
  (stdlib) under a Gmail adapter: **at-least-once**, with `cursor_at` deciding whether a restart delivers
  the mail that arrived while the watch was down or skips it (see
  [Usage: the watch](#usage-the-watch)). What it reports is an *arrival*, not a match. Never resolves;
  composes under `parallel [ … ]`.
- `gmail.provider(source = ...)` — provides the capability the tools require (`credential`) for the
  extent of a continuation, by resolving a `credentials.source` through the runtime on every ask. The
  provider holds no secret and keeps no cache: the runtime owns the token material, serves the stored
  access token while its recorded lifetime holds, and refreshes it against the stored token endpoint once
  due. The resolved token is a `string of private` — it flows only to the `Authorization: Bearer` header,
  never to a user-facing boundary.

The same stored Google credential serves the `google_calendar` package; give it a Gmail scope and both
read through one login. They also share their plumbing: the `Bearer` header, the 401/403 classification
and the authenticated **call** — one agent for GET, POST, PATCH and DELETE alike — live in the
**`google_common`** package, which both depend on. You never import it: it arrives as their transitive
dependency.

**Breaking in 0.5.0.** Optional arguments that used to default to `""` now default to `null`, and the
absent body part reports `null` rather than `""` — see [The body](#the-body) and
[Sending and labelling](#sending-and-labelling). A caller that passed `thread_id = ""` for a fresh send
now passes nothing (or `null`); a caller that tested `body.content_type == ""` now tests for `null`.
`http.auth_error` / `http.api_error` also carry a `status` and a `context` now (the stdlib's error
vocabulary v2), so a handler that reconstructs one has two more fields to fill — matching on the variant
is unchanged.

## Reading mail

`list_recent` is a `messages.list` for the ids followed by a `messages.get` per id, so a large
`max_results` is a proportional number of calls. Each message resource is read in Gmail's **full**
format: that one reply carries the headers, the snippet, the receive timestamp and the payload's part
tree, and the part tree is the only place an attachment is described.

An attachment's **bytes** never ride that reply. Google's contract makes a part's `data` and its
`attachmentId` mutually exclusive — when the id is present the content lives outside the message and
`data` is empty — so a mailbox holding a 10 MB PDF costs one part-tree entry per read, not a download.
Only `fetch_attachment` moves bytes.

### The body

`read_body` returns data, not a formatted string:

```katari
data message_body(content_type: string | null, text: string)
```

| `content_type` | `text` is |
| --- | --- |
| `"text/plain"` | the message's plain-text body, decoded |
| `"text/html"` | the message's **markup**, tags and all — it had no plain-text part |
| `null` | Gmail's short **snippet** preview — the message had neither part |

A great deal of real mail (a newsletter, a receipt, most mail a company sends) is HTML-only, and its
whole substance is in that markup. Returning it beats returning nothing, which would report a full
message as blank — but a caller has to be able to tell, so the type rides **beside** the text rather than
being announced inside it. Render the markup, strip it, or hand it to a model (which reads HTML fine);
the choice is the app's, and nothing here is truncated, so a long HTML mail is a long `text`.

```katari
import gmail

agent describe_body(id: string) -> string with gmail.credential | io | prelude.throw[http.api_failure | http.fetch_error | json.parse_error] {
  let body = gmail.read_body(id = id)
  match (body.content_type) {
    case "text/html" -> f"markup (${string.to_string(value = string.length(value = body.text))} characters)"
    case null -> f"no body part — Gmail's preview only: ${body.text}"
    case _ -> body.text
  }
}
```

(Before 0.3.0 this returned a bare `string`, with the HTML case announced by a line prefixed to the text.
A consumer reading the result as a string needs `body.text`. Before 0.5.0 the "neither part" case was
`""` rather than `null` — a media type is a string, so an empty one is a string a broken server could
actually send, and absence now has its own value instead of a convention a reader has to know.)

### Attachments

Every message already **lists** what it carries — knowing an attachment exists is one thing, asking for
its bytes is another:

```katari
data attachment(filename: string, content_type: string, size: integer, attachment_id: string)
```

`filename` is what the sender called it (`""` when it has none), `content_type` what the mail declares,
`size` the bytes Gmail reports, and `attachment_id` what fetches it — **with the message's own `id`**,
since an attachment id names a part of one particular message.

Listed is every part whose bytes Gmail holds behind an attachment id, which puts a mail's **inline**
images (the logo in an HTML signature) beside its real files. The mail itself draws no distinction
between them, so neither does this: the list reports what is there, and the app or the model decides
what is worth fetching.

`fetch_attachment` then returns a **`file`** — a handle whose bytes stay runtime-side. So the payload
never enters the conversation, a durable call record or a trace, and the handle can be passed straight to
any tool of yours that takes a file (an image viewer, for looking at a photo or a scanned invoice). The
file records the content type the mail declared for that part, which is what lets an image be recognised
as one downstream.

```katari
import gmail

@"The first attachment of a message, as a file, or null when it has none."
agent first_attachment(message: gmail.message) -> file | null with gmail.credential | io | prelude.throw[http.api_failure | files.malformed_base64 | http.fetch_error | json.parse_error] {
  match (array.get(target = message.attachments, index = 0)) {
    case null -> null
    case entry -> gmail.fetch_attachment(id = message.id, attachment_id = entry.attachment_id)
  }
}

@"Every PDF a message carries — the filter is the app's, over data the message already has."
agent pdf_attachments(message: gmail.message) -> array[file] with gmail.credential | io | prelude.throw[http.api_failure | files.malformed_base64 | http.fetch_error | json.parse_error] {
  array.flatten(target = for (let entry in message.attachments) {
    next if (entry.content_type == "application/pdf") {
      [gmail.fetch_attachment(id = message.id, attachment_id = entry.attachment_id)]
    } else {
      []
    }
  })
}
```

One call fetches one attachment. There is no bulk loop and no download-everything: the capability is a
named tool, and *when* to spend it is the caller's judgement.

Gmail sends the payload as **base64url** and may omit its padding, while the runtime's blob producer
(`files.from_base64`) validates the standard alphabet strictly — so the payload is translated before it
becomes a file. A payload that arrives corrupt throws `files.malformed_base64` rather than yielding a
broken file, which is why the tool's failure set is one wider than the other reads'.

## Sending and labelling

`send` writes an RFC 5322 message and posts it as `messages.send`. Compose the `subject` yourself — for a
reply, prefix `"Re: "` — and pass both threading values from the message you are answering: `thread_id`
files the reply in that Gmail conversation, and `in_reply_to` (the parent's `message_id`, its RFC822
`Message-ID`, **not** its `id`) writes the `In-Reply-To` / `References` headers so every mail client
threads it too. Both default to `null` — a fresh send — and an omitted one is a *smaller request* rather
than a blank field. (Before 0.5.0 they defaulted to `""`, which read as "not given" the same way; the
sentinel is gone because absence is `null` everywhere in this ecosystem, not because the behaviour
changed.)

```katari
import gmail

@"Reply inside the conversation a message came from."
agent reply(message: gmail.message, text: string) -> string with gmail.credential | io | prelude.throw[http.api_failure | http.fetch_error | json.parse_error] {
  gmail.send(
    to = message.sender,
    subject = f"Re: ${message.subject}",
    body = text,
    thread_id = message.thread_id,
    in_reply_to = message.message_id,
  )
}
```

`send` is plain text only, so it carries no attachments — there is no upload path here in either
direction. `modify_labels` covers filing and marking read; deletion is absent by design, so no tool in
this package can lose mail.

## Failure model

Three meanings, decided at three boundaries:

- **The credential needs a human** — never authorized, or its refresh is dead. This is **never an
  error**: the ask pauses the run on a `prelude.oauth.authorize` escalation (the admin console and
  `katari answer` render it as an authorization request), and completing the browser flow resumes the run
  where it stopped. Nothing to catch — a pause, not a throw.
- **`oauth.server_error`** (stdlib) — the token could not be resolved for a **transient** reason: a
  network error, or the token endpoint failing while refreshing. Thrown at the provider; retry it like
  any transport blip.
- **`http.api_failure = http.auth_error | http.api_error`** (stdlib) — the Gmail API's own failures,
  classified once by `http.classify_status` (which `google_common.classify_api_error` is the Google
  packages' name for). Both variants carry `status`, `context` and `message`:
  - `http.auth_error` — a 401/403: Google rejected the resolved access token (revoked before its stored
    expiry, or scoped too narrowly). Retrying the *same* token does not help, but **replaying the call
    does**: the re-run resolves the token afresh, the runtime refreshes the credential once its stored
    lifetime passes, and a credential the runtime cannot refresh parks the run as a re-authorization
    prompt. No app-owned escalation is needed.
  - `http.api_error` — any other non-2xx (a bad request, a missing message, a quota or server error).
    Read `status` to decide: 408 / 429 / 5xx are worth a backoff, anything else will fail identically
    until the argument changes.

  The pair is the **stdlib's**, not this package's: `google_calendar` and every other authenticated REST
  package classify into the same two variants, so one `replay` converter covers all of them at once.
  (This package used to declare its own `gmail_error = auth_error | api_error` — identical, down to the
  paragraph, to `google_calendar`'s. Before 0.5.0 the shared pair carried only a `message`, with the
  status folded into its prose; the number rides as a field now so a retry policy compares it.)

`fetch_attachment` adds `files.malformed_base64` for a corrupt payload — a bad reply, not a bad token.

## Setup: register the Google OAuth client, then log in

The runtime hosts the whole OAuth flow — the browser consent, the redirect callback, the token exchange,
storage and refresh. You register a Google OAuth client with the runtime once; programs then name the
credential and never see a token.

### 1. Create the OAuth client in the Google Cloud console

1. In [console.cloud.google.com](https://console.cloud.google.com), create (or select) a project and
   enable the **Gmail API** (APIs & Services → Library).
2. Configure the **OAuth consent screen**. While the app is in *Testing* status, add the Google account
   you will authorize as a test user — and note that Google expires a Testing app's refresh tokens after
   7 days, so **publish the app** (or use *Internal* on a Workspace domain) for a daemon that must keep
   running.
3. Create the client: APIs & Services → Credentials → Create Credentials → **OAuth client ID**, type
   **Web application**. Add the runtime's callback as an **authorized redirect URI**:
   `<public-url>/oauth/callback`, where `<public-url>` is the runtime's public base URL
   (`KATARI_PUBLIC_URL`; the local default is the runtime's own address, e.g.
   `http://localhost:3000/oauth/callback`).
4. Note the **Client ID** and **Client secret**.

### 2. Register the client with the runtime

In the admin console, open your project's **Credentials** page and press **Register client**:

| Field | Value |
| --- | --- |
| Name | `google` (whatever name the program passes to `credentials.oauth`) |
| Issuer | `https://accounts.google.com` |
| Authorization endpoint | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token endpoint | `https://oauth2.googleapis.com/token` |
| Client ID / Client secret | from the console (the secret is write-only — it is sealed and never echoed back) |
| Scopes | `https://www.googleapis.com/auth/gmail.modify` |
| Extra authorize parameters | `access_type=offline prompt=consent` |

`gmail.modify` is Google's read/write scope short of permanent deletion, and it authorizes every call
this package makes — list, get, `attachments.get`, send, label edits. An app that only ever reads can
register `gmail.readonly` instead and not call `send` / `modify_labels`; a narrower scope makes the
writes come back as `http.auth_error`, since Google reports "not permitted" as a 403.

The extra parameters matter for Google specifically: `access_type=offline` is what makes Google issue a
**refresh token** (without it the credential dies when the first access token expires), and
`prompt=consent` makes Google re-issue one on a later re-authorization instead of silently omitting it.

Sharing the credential with `google_calendar` means one client with **both** scopes registered
(`…/auth/gmail.modify` and `…/auth/calendar`); adding a scope to an existing credential requires logging
in again, since the stored grant is the one the consent screen issued.

### 3. Log in

Press **Log in** on the registered client's row and complete the consent screen — the runtime stores the
credential under the client's name. Or skip this: the first `credential` ask of a run **pauses** it on an
authorization escalation, which the admin console (and `katari answer`) render as the same login button;
authorizing resumes the run.

For several Google accounts, register the same client under several names (e.g. `google_work`) and pick
one per scope: `use gmail.provider(source = credentials.oauth(name = "google_work"))`.

## Usage: the tools

```katari
import gmail

agent recent(query: string) -> array[gmail.message] with io | prelude.throw[http.api_failure | oauth.server_error | env.missing_secret | http.fetch_error | json.parse_error] {
  use gmail.provider(source = credentials.oauth(name = "google"))
  gmail.list_recent(query = query, max_results = 5)
}
```

Hand `gmail.list_recent` / `gmail.read_body` / `gmail.fetch_attachment` / `gmail.send` /
`gmail.modify_labels` to an AI loop's tool list to let the model read and answer mail on its own. Each
carries a model-facing `@doc`, and the reads take no outside action — reading, labelling and *sending*
differ in consequence, so an app that gates anything gates the send.

## Usage: the watch

`gmail.watch` polls the mailbox and delivers each message that **arrives** matching `query`, oldest
first, to your `deliver_to` agent.

**What it reports is an arrival, not a match.** The mail already in the mailbox when the watch starts is
never delivered, however well it matches: a watch on `"is:unread"` over 200 unread messages stays silent
until the next mail lands. By the same token, mail that comes to match `query` *later* — an old message
you star, label or mark unread — is not delivered either. This is the right shape for hearing about
incoming mail, and the wrong one for a general "tell me whenever anything matches this search" trigger;
`list_recent` is what answers the latter.

Under the hood it is **`poll.subscribe`** — the stdlib's durable-subscription skeleton — with a Gmail
adapter, so it speaks that module's guarantees rather than its own.

What rides between ticks is a **watermark** (`poll.watermark(overlap = 1000)`), not a set of seen ids. The
subscription primes a cursor at the instant it starts and each tick searches `<query> after:<cursor>`, so
the backlog sits below the cursor and a message the cursor has passed can never come back. The cursor
advances by the **delivered messages' own** `internalDate` — Gmail's receive clock — never by the
poller's, so a slow tick, a long turn or a skewed clock cannot step over mail; the poller's clock enters
once, to cap how far one tick may carry the cursor, which can only hold it back. Its only companion
memory is the ids inside the 1000 ms overlap the fetch re-lists, which is what makes the second-granular
`after:` boundary safe in either direction — so the memory is **constant**, not the delivered history.

**Delivery is at-least-once.** A message reaches `deliver_to` one or more times, never zero: the commit
lands only after a delivery returns, so a crash or a throw in between re-delivers exactly that message on
the next tick. **Exactly-once is built at the destination**, by making the delivery idempotent — key it on
the message's `id`, which is the identity the subscription already promises is stable (label a message
rather than reply to it; upsert rather than insert).

**`cursor_at` decides what a restart does**, and there is no default — this is the consequential choice:

- `poll.resume(key = "…")` keeps the cursor in the **store**, so a runtime restart, a re-forked fiber and
  a `replay` re-run all resume from it: the mail that arrived while the watch was **down** is delivered,
  oldest first, on the tick after it comes back. This is what a persisted, re-forked or replayed watch
  wants. Give each watch a key of its own; two watches sharing one overwrite each other's cursor.
- `poll.fresh()` keeps it in the run alone: every fresh activation re-primes at that moment, and the
  downtime is skipped for good and silently. Choose it when re-priming *is* the semantics — a probe, a
  test, a watch whose backlog is worthless.

Within one activation a tick drains *every* page of its listing, so a watch that resumes after a long gap
catches up on the whole backlog that arrived meanwhile, oldest first.

The first poll is one `poll_interval_milliseconds` in, so a message arriving just after the watch starts
waits up to one interval — pending, not lost.

```katari
import gmail

@"Mark every arriving promotion read as it lands."
agent file_arrivals() -> never with io | store.get | store.set | prelude.throw[http.api_failure | oauth.server_error | env.missing_secret | http.fetch_error | json.parse_error] {
  use gmail.provider(source = credentials.oauth(name = "google"))
  // `deliver_to`'s parameter is named `value` — the prelude's primary-argument rule, which
  // `poll.subscribe` fixes for every watch built on it.
  agent mark_read(value: gmail.message) -> null with gmail.credential | io | prelude.throw[http.api_failure | http.fetch_error | json.parse_error] {
    gmail.modify_labels(id = value.id, remove = ["UNREAD"])
  }
  gmail.watch(
    query = "category:promotions",
    poll_interval_milliseconds = 300000.0,
    // Deliver what arrived while the process was down, rather than skipping it.
    cursor_at = poll.resume(key = "promotions-watch"),
    deliver_to = mark_read,
  )
}
```

Marking a message read is idempotent on its `id`, which is exactly the shape an at-least-once delivery
asks of its destination: a re-delivered message is removed from `UNREAD` a second time and nothing else
happens.

`deliver_to`'s effects flow out to the caller's handlers unchanged, so a delivery may itself call other
tools — read the body, fetch an attachment, reply, label. The watch never resolves, so it runs as a
`parallel [ … ]` arm or a standalone entry.

`watch` has **no built-in retry**: a poll failure — an http error, a rejected token, a `deliver_to` that
throws — propagates and kills the watch, exactly as an uncaught failure in any callee does. Resilience is
composed *around* it.

## Composition: resilience with `prelude.replay`

`prelude.replay` splits the retry **mechanism** from the failure **policy**. A `replay` *provider* re-runs
the rest of a block, but knows nothing about what counts as retriable: it catches exactly one request,
`replay.interrupted`, and re-runs after its delay. Deciding *which* failures become an `interrupted` is
your code — an ordinary `use handler` (a **converter**) between the provider and the block.

Under `oauth.token` the policy collapses to almost nothing, because re-authorization is no longer the
app's problem: replay the transient failures **and `http.auth_error`** (the re-run re-resolves the token,
and a credential that needs a human pauses on the runtime's own authorize escalation), and re-raise the
wiring defects. With `cursor_at = poll.resume(...)` the backoff costs nothing but latency — the cursor
outlives the failed activation, so the mail that arrived during it is delivered once the watch is back.

Two rules the converter must obey — both because a `throw` handler catches the **whole** throw union of
the block it guards, not a subset:

1. **Name every failure the block can raise.** Below that is `http.api_failure` (`http.auth_error |
   http.api_error`), plus `oauth.server_error` / `env.missing_secret` from the provider and
   `http.fetch_error` / `json.parse_error` from the calls — and whatever `deliver_to` itself throws.
   Because that pair is the **stdlib's**, the same converter covers a `google_calendar` arm added beside
   this one without naming a second package's error sum.
3. **Carry the cursor's store ops in the row.** `cursor_at = poll.resume(...)` is a `store.get` / a
   `store.set` per commit, so an agent wrapping the watch declares them (`with io | store.get | store.set
   | …`) — they resolve wherever the app installs its `store.workspace` / `store.scope`.
2. **Reconstruct a re-raised failure** rather than re-throwing the match scrutinee: `prelude.throw(error
   = error)` would widen the residual back to the whole union, so each propagated case rebuilds its own
   value.

```katari
import gmail

@"The watch under one converter: replay the transient failures and the rejected tokens, propagate the defects."
agent file_arrivals_resiliently() -> never with io | store.get | store.set | prelude.throw[env.missing_secret] {
  agent mark_read(value: gmail.message) -> null with gmail.credential | io | prelude.throw[http.api_failure | http.fetch_error | json.parse_error] {
    gmail.modify_labels(id = value.id, remove = ["UNREAD"])
  }
  // MECHANISM: re-run the block after a backoff (100ms, doubling, capped at a minute).
  use replay.forever(initial_delay_milliseconds = 100.0, factor = 2.0, max_delay_milliseconds = 60000.0)
  // POLICY: names every failure the block can throw, then dispatches.
  use handler {
    request prelude.throw(error: http.api_failure | oauth.server_error | env.missing_secret | http.fetch_error | json.parse_error) -> never {
      match (error) {
        // A credential that is not registered at all is a wiring defect: no backoff fixes it.
        case env.missing_secret(key => key, message => message) -> { prelude.throw(error = env.missing_secret(key = key, message = message)) }
        // Everything else is transient, a rejected token included — the re-run re-resolves it.
        case _ -> { replay.interrupted(failure = error) }
      }
    }
  }
  // INSIDE the replay scope, so a re-run RE-ENTERS the provider and its next `credential` ask
  // re-resolves through the runtime.
  use gmail.provider(source = credentials.oauth(name = "google"))
  gmail.watch(
    query = "category:promotions",
    poll_interval_milliseconds = 300000.0,
    // The cursor lives in the store, so the backoff loses nothing: the re-run resumes where the failed
    // activation stood and delivers what arrived while it was backing off.
    cursor_at = poll.resume(key = "promotions-watch"),
    deliver_to = mark_read,
  )
}
```

An `http.api_error` replay re-runs the same call past a transient fault; an `http.auth_error` replay is a
**token re-resolution** whose end state, when a human really is needed, is the runtime's own re-authorization
pause. Until the rejected token's stored lifetime passes — at most about an hour — the replays re-see the
same token and fail again, which is why a capped backoff like `replay.forever`, not `replay.immediate`, is
the right mechanism here.
