# Email-Guard Specification

Version: 1.3.0
Status: stable. Reference implementation:
[email-guard-core-php](https://github.com/Emailsherlock1/email-guard-core-php)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## 1. Scope

Email-Guard answers one question at form-submit time: should this form accept
the email address the visitor just typed? It is a guard for signups and
checkouts, not a list-cleaning tool and not an MTA.

This document specifies:

- the **local checks** every implementation runs without network access
  (section 4)
- the **remote check** against the EmailSherlock Verify API and how its
  response maps into the model (section 5)
- the **decision model** that turns facts into an action (section 6)
- the **result object** an implementation returns (section 7)
- the **conformance vectors** that pin all of the above across languages
  (section 8)
- the **data formats** for bundled artifacts (section 9)

It deliberately does not specify framework integration surface (constraints,
form options, plugin hooks). That belongs to each integration.

## 2. Terminology

| Term | Meaning |
|------|---------|
| guard | A configured instance of an implementation, ready to check addresses |
| verdict | The classification of an address: one of the values in section 6.1 |
| action | What the form should do: `allow`, `deny`, or `review` |
| reason | A machine-readable code explaining the verdict (section 6.2) |
| local check | A check that runs in-process with bundled data, no network |
| remote check | A call to `POST /v1/verify/single` on the Verify API |
| degraded | The guard wanted the remote check but the API was unusable |

## 3. Input normalization

Before any check, an implementation MUST:

1. Trim ASCII whitespace (space, tab, CR, LF) from both ends of the input.
2. Use the trimmed string as the address under test. The domain part is
   compared case-insensitively everywhere; implementations SHOULD lowercase
   it once and reuse that. The local part keeps its original case (it is sent
   to the API as typed).

No other rewriting. The guard never "fixes" an address.

## 4. Local checks

Local checks run in order. The first check that produces a verdict ends the
local phase; later checks and the remote check are skipped. If no local check
produces a verdict, the address proceeds to the remote check (section 5).

Order: syntax (4.1), then reserved TLD (4.2), then disposable snapshot (4.3).

Local checks can only produce negative verdicts (`invalid`, `disposable`).
Passing all local checks proves nothing about deliverability, so the local
phase alone never yields `valid`. Without an API key the final verdict for a
clean address is `unknown`.

### 4.1 Syntax

A pragmatic profile, not full RFC 5322. The target is text a human typed into
a form field. Quoted local parts, address literals, comments, and folding
whitespace are legal in mail headers but never legitimate signup input, so
this profile rejects them. Where this profile and the RFC disagree, the
profile wins and a conformance vector pins the behavior.

**Structural rules.** These always apply:

- The address MUST contain exactly one `@`.
- The local part (before `@`) and the domain part (after `@`) MUST both be
  non-empty.

**Profile rules.** These apply per part, and are skipped for a part that
contains any non-ASCII character (see "International addresses" below):

Local part:

- 1 to 64 characters.
- Allowed characters: `A-Z a-z 0-9` and `! # $ % & ' * + - / = ? ^ _ ` { | } ~ .`
- MUST NOT start or end with `.` and MUST NOT contain `..`.
- Quoted strings (`"jane doe"@…`) are not part of the profile; the `"` and
  space characters are simply not allowed, which rejects them.

Domain part:

- 4 to 253 characters.
- At least two labels separated by `.`. A trailing dot means an empty final
  label and is invalid.
- Each label: 1 to 63 characters from `A-Z a-z 0-9 -`, not starting or
  ending with `-`.
- The final label (the TLD): at least 2 characters, not all digits. This
  also rejects address literals like `[192.168.0.1]`.

A violation produces verdict `invalid` with reason `bad_syntax`.

**International addresses.** If a part contains any code point above U+007F,
its profile rules are skipped and that part is treated as syntactically
acceptable. Rationale: the guard blocks what it can prove is junk. A strict
ASCII profile would reject internationalized addresses (RFC 6531) that real
people use; proving them undeliverable is the API's job, not a regex's.

### 4.2 Reserved TLD

Domains under reserved names can never receive public mail. The check runs on
the lowercased domain part and produces verdict `invalid` with reason
`reserved_tld` when one of these matches:

- the final label is a **reserved TLD**: `test`, `example`, `invalid`,
  `localhost`, `local`, `onion`, `alt`, `internal`
- the domain equals, or is a subdomain of, a **reserved domain**:
  `example.com`, `example.net`, `example.org`

Subdomain matching means suffix-with-dot: `mail.example.org` matches
`example.org`; `exampleshop.com` does not.

Sources: [RFC 2606](https://www.rfc-editor.org/rfc/rfc2606) /
[RFC 6761](https://www.rfc-editor.org/rfc/rfc6761) (`test`, `example`,
`invalid`, `localhost`, the `example.*` domains),
[RFC 6762](https://www.rfc-editor.org/rfc/rfc6762) (`local`),
[RFC 7686](https://www.rfc-editor.org/rfc/rfc7686) (`onion`),
[RFC 9476](https://www.rfc-editor.org/rfc/rfc9476) (`alt`), and the
[ICANN board resolution of 2024-07-29](https://www.icann.org/en/board-activities-and-meetings/materials/approved-resolutions-special-meeting-of-the-icann-board-29-07-2024-en)
(`internal`).

This list is part of the spec. Adding a name is a minor version bump,
removing one is a major bump.

### 4.3 Disposable snapshot

Implementations bundle a versioned snapshot of disposable-mail domains
(format in section 9.1). The check compares the lowercased domain part
against the snapshot with **exact matching**: the domain matches an entry or
it does not. No suffix matching.

Exact matching is deliberate: the Verify API matches its live list the same
way, and the local check MUST NOT block an address the API would let pass.
A subdomain of a disposable provider falls through to the remote check.

A match produces verdict `disposable` with reason `disposable_provider`.

The snapshot is a point-in-time export of the API's live list. It catches the
bulk of disposable traffic for free; the live list behind an API key catches
the rest.

## 5. Remote check

The remote check runs when all three hold:

1. no local check produced a verdict,
2. an API key is configured,
3. the transport is able to try (no open circuit breaker, if implemented).

The implementation sends `POST /v1/verify/single` with the normalized address
and the configured key, and applies a total timeout of `timeout_ms`
(default 800). The endpoint contract, all response fields, and authentication
are specified by the published OpenAPI document at
`https://emailsherlock.com/v1/openapi.json`; this spec does not duplicate it.

### 5.1 Response mapping

These response fields feed the decision model:

- `result` is the verdict, verbatim. The API enum (`valid`, `invalid`,
  `catch_all`, `disposable`, `role`, `unknown`) is identical to the verdict
  enum in section 6.1.
- `reason`, when non-null, is appended to the result's reason list.
- `relay` (bool, added in 1.2.0) is the `relay` signal (section 6.5). It
  overlays the verdict instead of replacing it: an address can be both `valid`
  and `relay`. It influences the action only when the integrator opts in via
  `block_on` / `review_on`; otherwise it is informational.

Everything else in the response (`score`, `deliverable`, `relay_provider`, the
`domain` object) is informational. Implementations SHOULD expose the raw
response to callers but MUST NOT let it influence the action beyond the `relay`
rule in section 6.5.

An API answer of `result: "unknown"` (greylisting, SMTP timeout, a deferred
batch) is a normal verdict and flows through the decision model like any
other. It is not degradation.

### 5.2 Failure semantics

Any of these makes the check **degraded**: connection failure, total timeout
exceeded, or any non-2xx HTTP status. There is no retry at check time; one
form submit gets at most one API call.

A degraded check yields verdict `unknown`, sets `degraded: true` on the
result, and carries `api_unavailable` as its first reason. The action is then
decided by `fail_open` alone (section 6.4).

When the implementation can attribute the failure, it appends one granular
reason after `api_unavailable` so telemetry can tell "the API was slow" apart
from "the API errored" apart from "the API was unreachable":

| Granular reason | Cause |
|-----------------|-------|
| `api_timeout` | the total or connect timeout budget was exceeded |
| `api_unreachable` | DNS, TLS, or connection failure before any HTTP status |
| `api_http_5xx` | the API answered a 5xx status |
| `api_http_4xx` | the API answered a 4xx status (bad key, rate limit, …) |
| `api_bad_response` | a 2xx answer whose body was not the expected JSON |

The granular reason is best-effort: a transport that cannot separate a timeout
from a connection failure (a PSR-18 client that surfaces both as one exception,
for example) MAY omit it and leave `api_unavailable` standing alone.
`api_unavailable` is present on every degraded check, so a consumer that only
counts outages never has to special-case the granular set. The HTTP status
code is deliberately not part of the reason name, which keeps the reason set
bounded; an implementation MAY expose the code elsewhere (the raw response, a
log line).

Implementations SHOULD cache recent API responses keyed by address (the API
itself answers from cache where it can) and MAY add a circuit breaker that
skips the transport for a short window after repeated failures. Both produce
the same observable states as above: a circuit-breaker skip is a degraded
check.

## 6. Decision model

### 6.1 Verdicts

```
valid | invalid | disposable | role | catch_all | unknown
```

Same strings as the API's `result` field. `unknown` covers three situations:
clean address with no API key, API answered `unknown`, or degraded check.

### 6.2 Reasons

Reasons from local checks:

| Reason | Produced by |
|--------|-------------|
| `bad_syntax` | section 4.1 (also used by the API's syntax layer) |
| `reserved_tld` | section 4.2 (local only) |
| `disposable_provider` | section 4.3 (also used by the API) |
| `api_unavailable` | section 5.2 (local only) |
| `api_timeout` | section 5.2 (local only) |
| `api_unreachable` | section 5.2 (local only) |
| `api_http_4xx` | section 5.2 (local only) |
| `api_http_5xx` | section 5.2 (local only) |
| `api_bad_response` | section 5.2 (local only) |

Reasons from the API arrive in the `reason` response field and pass through
verbatim (`no_mx`, `mailbox_accepts`, `mailbox_not_found`, `role_address`,
`catch_all_domain`, `greylisted`, `smtp_timeout`, `smtp_unreachable`,
`verification_pending`, and any value a future API version adds).
Implementations MUST pass unrecognized reason strings through rather than
reject them.

When a signal is active (section 6.5), its name is appended to the reason
list. The only signal so far is `relay`.

### 6.3 Configuration

| Key | Type | Default | Meaning |
|-----|------|---------|---------|
| `api_key` | string or null | null | null disables the remote check |
| `block_on` | list of verdicts and signals | `["invalid", "disposable"]` | verdicts (and signals, section 6.5) that deny |
| `review_on` | list of verdicts and signals | `[]` | verdicts (and signals, section 6.5) that flag for review |
| `fail_open` | bool | `true` | what a degraded check does |
| `timeout_ms` | int | `800` | total budget for the remote check |

Policy belongs to the integrating developer. The defaults block what is
provably junk and let everything debatable (role addresses, catch-all
domains, unknowns) through. `review` is for flows that have a second gate:
hold the signup, require email confirmation, queue for a human.

`block_on` / `review_on` accept verdicts (section 6.1) and signals
(section 6.5). An unknown token is inert: it matches nothing and is not an
error, so an integrator can list `relay` against an older core library without
breaking.

### 6.4 Action resolution

```
if degraded:
    action = fail_open ? allow : deny
else:
    blocked  = verdict in block_on  or (relay and "relay" in block_on)
    reviewed = verdict in review_on or (relay and "relay" in review_on)
    action = blocked ? deny : reviewed ? review : allow
```

`relay` is the boolean signal from section 6.5 (false unless the remote check
set it). Three rules fall out of this and are pinned by vectors:

- `deny` beats `review`: a verdict or signal listed in both lists denies.
- A signal only acts when listed: `relay` defaults to inert, so an address on
  a relay service passes unless the integrator opts in.
- Degradation bypasses `block_on` and `review_on` entirely. Listing
  `unknown` in `block_on` is a statement about *answered* unknowns
  (greylisting, missing key); it MUST NOT turn an API outage into a wall of
  rejected signups. Operators who want that behavior set
  `fail_open: false`, which says exactly that.

Fail-open is the default because the cost asymmetry is stark: a blocked
legitimate customer is worth more than a leaked junk signup.

### 6.5 Signals

A *signal* is a boolean property of the remote response that an integrator can
place in `block_on` / `review_on` next to verdicts. Unlike a verdict, a signal
does not replace the verdict: the address keeps its `valid` / `catch_all` / …
classification and may also carry the signal.

| Signal | Source | Meaning |
|--------|--------|---------|
| `relay` | response `relay: true` | the address sits on a relay / forwarding / alias-masking service (SimpleLogin, Apple Hide My Email, Cloudflare Email Routing, …). It is deliverable and reaches a real person, just masked. `relay_provider` names the service and is informational. |

Rules:

- A signal influences the action only when the integrator lists it (section
  6.4). By default `relay` does nothing.
- A signal is **active** when it is true and listed in `block_on` or
  `review_on`. When a signal is active, its name is appended to the result's
  reasons (so telemetry, section 11, and the dashboard can attribute the
  decision), even if a verdict independently produced the same or a stronger
  action.
- Signals are produced only by the remote check. Local checks and degraded
  checks never set them (`relay` is false), so a signal never fires without an
  API answer.

## 7. Result object

A check returns at least these four fields (naming may follow language
convention, e.g. `apiCalled` in JS; the conformance harness maps them):

| Field | Type | Meaning |
|-------|------|---------|
| `verdict` | string | section 6.1 |
| `action` | string | `allow`, `deny`, or `review` |
| `reasons` | list of strings | ordered, possibly empty |
| `degraded` | bool | true only when a wanted API call failed |

`degraded` is false when no key is configured: the absence of a key is a
configuration, not an outage.

Implementations MAY add fields (the raw API response, timing) but MUST NOT
change the meaning of these four.

## 8. Conformance

The files under [vectors/](vectors/) are the normative test set. Each file
holds vectors for one category; [vectors/schema.json](vectors/schema.json)
describes the format.

A vector provides:

- `input.email`: the raw form input, untrimmed.
- `input.config`: partial configuration; omitted keys take the defaults from
  section 6.3.
- `input.api`: the mocked transport. `null` when no key is configured,
  the string `"unavailable"` to simulate timeout or 5xx, or an object that
  the mock returns as the parsed API response body.
- `expected`: `verdict`, `action`, `reasons`, `degraded`, plus `api_called`
  (whether the implementation hit the transport at all).

A conforming implementation:

1. loads every `vectors/*.json` file,
2. installs [vectors/fixtures/disposable-snapshot.json](vectors/fixtures/disposable-snapshot.json)
   as its bundled snapshot,
3. for each vector, builds a guard from the config, wires the transport mock,
   checks the email, and asserts all five expected values.

All vectors MUST pass. `expected.api_called` is part of the assertion: it
proves local short-circuits never burn an API call.

## 9. Data formats

### 9.1 Disposable snapshot

A JSON document, published versioned (one source, consumed by every
core library per release; no live fetching):

```json
{
  "format": "email-guard-disposable/1",
  "version": "2026.06.11",
  "generated_at": "2026-06-11T04:00:00Z",
  "count": 3,
  "domains": ["10minutemail.com", "guerrillamail.com", "mailinator.com"]
}
```

- `format`: literal `email-guard-disposable/1`; a breaking change to this
  shape bumps the suffix.
- `version`: calendar version of the export, `YYYY.MM.DD`.
- `domains`: lowercase, ASCII (IDNs in punycode), sorted, unique. `count`
  MUST equal the array length; consumers SHOULD verify it on load.

### 9.2 Reserved names

The reserved TLD and reserved domain lists live in section 4.2 of this
document and in the conformance vectors. They are small and change on the
timescale of RFCs, so they ship as part of each core library release rather
than as a separate artifact.

## 10. Versioning

This spec follows [SemVer](https://semver.org).

- **Major**: any change that flips an existing vector's expected output, or
  removes a reserved name, verdict, action, or result field.
- **Minor**: new vectors, new reserved names, new optional config keys,
  clarifications that pin previously unspecified behavior.
- **Patch**: editorial fixes that change no behavior.

Core libraries state which spec version they implement and embed the vector
set of that version in their test suite.

## 11. Decision telemetry (optional, added in 1.1.0)

An integration MAY report each decision back to EmailSherlock so the account
owner can see what the guard blocks. This is the data unlock behind the guard
dashboard: without a key the guard blocks silently, with a key you see what it
blocked. Reporting is entirely optional and never changes a decision.

### 11.1 Privacy: domain only, never the address

A telemetry event carries the **domain**, never the local part and never the
full address. A bare domain (`mailinator.com`) is not personal data; the
local part is. The analytics this feeds (top blocked domains, action/reason
counts, degraded rate) need nothing more, so an event MUST NOT include the
address. An implementation that cannot extract a domain (syntax failed before
an `@`) sends `domain: null`.

### 11.2 Event shape

```json
{
  "domain": "mailinator.com",
  "verdict": "disposable",
  "action": "deny",
  "reasons": ["disposable_provider"],
  "degraded": false,
  "source": "local",
  "integration": "symfony-bundle",
  "lib_version": "0.1.2",
  "spec_version": "1.0.0"
}
```

- `domain`: lowercased domain part, or null. Never the full address.
- `verdict`, `action`, `degraded`: as in sections 6 and 7.
- `reasons`: the result's reason list, verbatim.
- `source`: `local` if a local check decided it, `remote` if the API did.
- `integration`, `lib_version`, `spec_version`: optional provenance strings.

### 11.3 Reporting semantics

- **Fire-and-forget, fail-silent.** Reporting MUST NOT delay or change the
  form submit. In PHP, report after `fastcgi_finish_request`. A failed report
  is dropped, never surfaced to the visitor.
- **Batched.** Events SHOULD be sent in batches, not one request per submit.
- **Key-gated.** Reporting runs only when an API key is configured. No key,
  no telemetry.
- **Transport.** `POST /v1/guard/events` with `{ "events": [ ... ] }`. The
  endpoint is free (no credits), returns `202 { accepted, rejected }`, and
  drops malformed items rather than rejecting the whole batch. The full
  request/response contract lives in the published OpenAPI document, not here.

### 11.4 No conformance vectors

Unlike the decision model, telemetry has no conformance vectors: it is an
outbound side effect, not a verdict, so there is nothing for the cross-language
vector set to pin. The endpoint contract is covered by the server's own tests.
