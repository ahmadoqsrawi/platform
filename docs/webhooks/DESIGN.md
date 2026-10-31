# Outbound webhook subscriptions

## Status: Design (rev 4)

This document is the design proposal for the outbound webhook feature
discussed in Platform-Collective/platform discussion #236 and issue #33.
No code lives on `feature/webhooks-design`; this doc is the whole PR.

The feature lets a workspace Owner subscribe an outbound HTTP endpoint
to Tx events in the workspace. Method, URL, headers and body are all
templated; named secrets go through a dedicated `{{ secrets.<name> }}`
namespace that never renders into logs, previews or audit rows. One
target kind (`generic`) handles every receiver via templates plus
built-in presets for `apprise`, `slack`, `discord` and bare `generic`.
Authorization is enforced in a plugin-contributed permission middleware
placed BEFORE storage writes, same slot as `guestPermissions`.
Destination / filter changes are co-signed by the subscription's
stored owner, not any current workspace Owner. Secrets live in an
AEAD-encrypted blob store whose AAD is reconstructed from the
authenticated context on every decrypt, not read from the stored row.
A durable outbox is written in the SAME storage transaction as the
source mutation, with an explicit journal recovery design when a single
cross-domain transaction is not available. Audit payloads live outside
ordinary Tx history. Signatures use a formal versioned header grammar
with repeated headers during key rotation and an immutable per-
delivery payload. Test events return status + metadata only by
default; body opt-in is an explicit credential-disclosure
authorization.

### Revision history

- **Rev 1** (`7831133506`): first draft. Separate `apprise` target
  kind; secret stored in the body.
- **Rev 2** (`cc250b7575`): collapsed `apprise` into a preset of
  `generic`; added named `{{ secrets.<name> }}` namespace with
  redaction; disabled redirects; HMAC v=2 covered timestamp +
  delivery id.
- **Rev 3** (`c46a6e4d98`): rebuilt around MichaelUray's rev-2
  review. AEAD secrets with HKDF-derived DEKs; destination binding
  replacing single-host comparison; per-attempt eligibility re-check;
  outbox job collection with full dedup key; placeholder-first
  redaction.
- **Rev 4** (this revision): addresses MichaelUray's rev-3 review.
  8 must-fix items and 9 should-clarify items reshape Authorization,
  Secrets, Delivery, Signing, Audit and Restore. Three new open
  questions settled with his answers.

## Open questions, settled

| # | Question | Answer (from rev-3 review) |
|---|---|---|
| 1 | Service naming | `services/webhooks/pod-webhooks` |
| 2 | Settings tab | Workspace settings > Webhooks, Owner-only |
| 3 | Expression language | None in v1 |
| 4 | Second-Owner for destination changes | **Opt-in workspace policy, off by default**. Re-entering every referenced secret stays the default protection; many self-hosted workspaces have a single Owner, so a mandatory second approval would block them. |
| 5 | `WEBHOOKS_MASTER_KEY` | **Required at service start**. The service refuses to start without it; a one-line key-generation command in the deployment docs keeps the self-hoster path easy. |
| 6 | `replayHorizonSeconds` | **Default 300 s** (matches Stripe / Slack); subscription-level override allowed for sensitive receivers. |

## Why outbound

Teams run external integrations today by polling Huly's API from an
external worker with a long-lived token. Hoisting the outbound into
the platform removes the external component, keeps the subscription
set in the same backup as the workspace, and lets a non-engineer admin
maintain it from workspace settings.

## Scope

In scope for v1:

- Issue events only: `IssueCreated`, `IssueUpdated`, `IssueStatusChange`.
- Workspace Owner-managed subscriptions, per-workspace.
- Project and status filters per subscription (explicit project OR
  explicit all-projects marker).
- One target kind `generic` (templated method / URL / headers / body)
  with built-in presets for `apprise`, `slack`, `discord` and bare
  `generic`.
- Named, write-only secrets with authenticated encryption; AAD
  reconstructed from authenticated context on every decrypt.
- SSRF protection: public-only by default; IP-literal normalization;
  every A/AAAA validated per attempt; connection pinned to the
  validated IP; proxy transport pinned to that IP as well; no
  implicit proxy; redirects disabled.
- Signatures use a versioned grammar with repeated headers during
  key rotation; payload is immutable per `deliveryId`.
- Delivery outside the transactor hot path: a Kafka-group consumer
  that acknowledges after durable outbox handoff (NOT after terminal
  HTTP), subscription-wide ordered leases over the outbox, bounded
  retry + quarantine, backlog + fan-out quotas.
- Audit payloads live outside ordinary Tx history (dedicated encrypted
  blob store); audit reads require current source access.
- Restore ALWAYS pauses subscriptions, regardless of UUID or key
  recovery.
- Per-subscription audit history and a Send-test-event action; test
  returns status + approved metadata only by default.

Explicitly out of v1: Doc + Membership events, per-user personal
subscriptions, cross-workspace fan-out, expression language, inbound
HTTP (#33).

## Coordination with #33 (inbound) and #238 (workflow-rules)

Three parallel features touch "events across the Huly boundary":

- **Outbound (this)** — Huly to external.
- **#33 (inbound)** — external to Huly. Draft at
  thomas-bouvier/platform:feat/webhook.
- **#238 (workflow-rules)** — project-to-project status mirror,
  internal only. In review.

Shared conventions (open to counter-proposal from @thomas-bouvier):

| Concern | Convention |
|---|---|
| Service layout | `services/<feature>/pod-<feature>` for both; no runtime cross-dependency. |
| Secret storage | The AEAD scheme in [Secrets](#secrets); shared by both. |
| UI home | Workspace settings; admin-only; NOT the per-user Integrations category. |
| Rate limits | Sliding window keyed on `(workspace, subscriptionId)` for outbound; analogous for inbound. |
| Audit storage | Dedicated encrypted blob store, outside ordinary Tx history. |
| Permission to manage | Workspace Owner for both; opt-in via Settings > Configure like #238. |

## Shape of a subscription

```ts
interface WebhookSubscription extends Doc {
  // Workspace-scoped. Doc.space is Ref<Space> per Platform's typing;
  // the subscription's space is the workspace's dedicated settings
  // space. The service NEVER routes by this field; routing is by
  // authenticated WorkspaceUuid in the service context.
  space: Ref<Space>

  name: string

  // The account whose source-read permissions gate every delivery
  // and every management action that targets this subscription.
  // AccountUuid (not Ref<Account>; Account is UUID-based, not a Doc).
  // Immutable after create.
  owner: AccountUuid

  // Target config (templated method/URL/headers/body + binding).
  target: WebhookTarget

  // Monotonic revision. Incremented by the service on every
  // destination-changing save. Client Tx attempts to set this are
  // rejected by the permission middleware.
  revision: number

  // Named-secret metadata. No `lastFour`: a short secret would
  // leak fully. Only name + rotation time are exposed.
  secretsMeta: Array<{ name: string, lastRotatedAt: Timestamp }>

  triggers: WebhookTrigger[]

  retryPolicy?: WebhookRetryPolicy

  // Soft-off switch. Re-checked on every attempt.
  enabled: boolean

  // Opt-in: receiver response body stored in audit (default false).
  // Reset to false on every destination-changing save.
  storeResponseBody: boolean

  // Observability. Service-maintained; immutable from clients.
  lastDeliveryAt?: Timestamp
  lastDeliveryStatus?: 'ok' | 'error' | 'skipped'
  lastDeliveryHttpCode?: number
  lastDeliveryError?: string
}

interface WebhookTarget {
  preset?: 'apprise' | 'slack' | 'discord' | 'generic'
  method: 'POST' | 'PUT'
  urlTemplate: string
  headerTemplates: Array<{ name: string, valueTemplate: string }>
  bodyTemplate?: string
  useMinimalBody?: boolean
  contentType?: string

  // Configuration digest, computed over APPROVED CONFIG (the
  // templates, method, content type, URL template, header names and
  // value templates, body template, and the sorted list of secret
  // names each template references). NOT over rendered values.
  // See "Binding".
  configDigest: string
  configDigestAlg: 'sha256'
  boundAt: Timestamp
}

interface WebhookTrigger {
  event: 'IssueCreated' | 'IssueUpdated' | 'IssueStatusChange'

  // Explicit project selection OR explicit all-projects marker.
  // Rev 3 made this explicit; rev 4 keeps it.
  projectScope: { kind: 'project', projectId: Ref<Project> }
                | { kind: 'allProjects' }

  statusFilter?: {
    on: 'previous' | 'new'
    statuses: Ref<IssueStatus>[]
  }
}

interface WebhookRetryPolicy {
  maxRetries: number           // default 5
  initialBackoffMs: number     // default 2000
  backoffFactor: number        // default 2
  jitterMs: number             // default 500
  timeoutMs: number            // default 10000 (hard-capped at 30000)

  // Per-subscription override of the global replay horizon advertised
  // to receivers. Default 300 s (set globally); a subscription with
  // a sensitive receiver can lower this; a subscription cannot raise
  // it past a service-wide ceiling.
  replayHorizonSeconds?: number
}
```

## Authorization

### Placement (fixing MichaelUray's point #1)

Rev 3 proposed a trigger. Rev 4 contributes a plugin **permission
middleware** that runs BEFORE storage writes, same slot as
`guestPermissions` in `foundations/server/packages/middleware/`.
Rationale: `TriggersMiddleware.tx` calls `provideTx` BEFORE
`processDerived`, and trigger exceptions are caught, so a direct
`TxUpdateDoc` from a non-Owner lands in storage even if a trigger
would have rejected it. A permission middleware refuses the Tx before
`provideTx` and nothing is persisted.

The middleware's intercept list:

- **Create / update / remove `TWebhookSubscription`**: requires
  current `AccountRole.Owner` on the workspace of the targeted doc.
  Compound transactions (TxApplyIf, TxCreateDoc wrapping other CUD
  via `collection`) are walked recursively so a non-Owner cannot
  smuggle a subscription edit through a collection write.
- **Changes to the subscription's `owner` field**: rejected
  unconditionally. `owner` is set at create and never mutated.
- **Changes to `revision`, `target.configDigest`, `target.boundAt`,
  `lastDelivery*`, `secretsMeta`**: rejected from any client Tx;
  service-maintained only.
- **Destination-changing fields** (`target.method`, `target.urlTemplate`,
  `target.headerTemplates`, `target.bodyTemplate`,
  `target.useMinimalBody`, `target.contentType`): must come from the
  **stored owner** of the subscription OR carry a second-Owner
  authorization token when the workspace policy requires it. See
  "Owner-authorized destination changes" below. Fixes point #3.
- **Create / update / remove of `TWebhookSecret`**: rejected from
  any direct client Tx. The service's management RPC is the only
  write path (see [Secrets](#secrets)).
- **Query `TWebhookDeliveryAttempt`**: filtered by the middleware
  so a caller sees only rows whose referenced source doc the caller
  can currently read AND whose subscription owner is still eligible.
  Rows that fail either check surface as `{reason, count}` buckets,
  no identifiers. Fixes point #4.

The middleware is unit-tested with the same shape as
`guestPermissions`.

### Owner-authorized destination changes (fixing point #3)

Rev 3 required re-entering secrets on destination changes but did not
pin which Owner could authorize the change. A second workspace Owner
without source access to the subscription's source project could
redirect that subscription's delivery to their own receiver.

Rev 4 pins destination-changing saves to the subscription's **stored
owner**:

- The management RPC accepts a destination-changing save ONLY when
  the authenticated caller is the subscription's stored `owner`.
  Any current Owner can disable or delete a subscription (that is
  containment, not configuration), but only the stored owner can
  modify its destination/filter/body/secrets.
- A workspace that wants a second-Owner co-sign enables the
  `requireSecondOwnerForDestinationChanges` policy (opt-in; off by
  default per rev-3 open question #4). With the policy on, a
  destination-changing save must carry a short-lived authorization
  token issued by a different current Owner for that exact
  `(subscriptionId, proposedRevision, proposedConfigDigest)` triple.
- A stored owner who loses workspace membership can no longer save
  a destination change; the management RPC refuses with
  `errorReason: 'ownerIneligible'`. Enable/disable/delete still
  work for current Owners because they do not require source
  access.

Transferring ownership is a one-time management action gated on
BOTH the current stored owner AND the target account being a
current workspace Owner.

### Delivery eligibility (per attempt and per retry)

Right before every outbound HTTP call, including every retry and
every test event, the delivery service re-checks:

- Plugin is enabled in Settings > Configure (`PluginConfiguration.enabled === true`).
- Subscription doc still exists AND `enabled === true`.
- Subscription owner is still a workspace member AND still holds
  `AccountRole.Owner`.
- Subscription owner currently has read access to the source doc.
- Target project exists, is not archived, is not restricted.
- Job's `revision` and `configDigest` both match the subscription's
  current values (binding guard; see [Binding](#binding)).
- Rendered destination passes SSRF checks (independent from the
  configDigest, which hashes the approved config, not the rendered
  values; fixes point #10).

Failures drop the attempt with the matching `errorReason`. The job
transitions to `failed` for permanent reasons, `pending` with
exponential backoff for transient ones.

### Tests and previews (fixing point #3)

- A test or preview requires the caller to be the subscription's
  **stored owner** OR, when `requireSecondOwnerForDestinationChanges`
  is on, carry a token from a different current Owner.
- The chosen source doc for the preview must be readable by the
  caller AND by the stored owner. If either lacks read access, the
  action is refused with `errorReason: 'previewSourceUnauthorized'`.

## Secrets

Secrets are a separate concept from the subscription body and never
land in persisted Tx bodies.

### Storage

```ts
TWebhookSecret extends TDoc {
  space: Ref<Space>              // workspace settings space
  subscription: Ref<WebhookSubscription>
  name: string                   // matches secretsMeta[i].name
  masterKeyId: string            // which master key wraps this row
  dekVersion: number             // which HKDF derivation of that master key
  ciphertext: Buffer             // AES-256-GCM output: encrypted value only
  nonce: Buffer                  // 96-bit, random per encrypt
  tag: Buffer                    // 128-bit GCM auth tag
  lastRotatedAt: Timestamp
}
```

**AAD reconstruction (fixing point #6).** The row stores ciphertext,
nonce, tag, master key id, and DEK version. It does NOT store the
workspace uuid, subscription id or secret name as "authoritative"
inputs to decryption. On every decrypt, the AAD is reconstructed as:

```
aad = authenticatedWorkspaceUuid
   || requestedSubscriptionId
   || requestedSecretName
   || masterKeyId
   || dekVersion
```

where `authenticatedWorkspaceUuid` comes from the service's own
authenticated transactor connection, `requestedSubscriptionId` and
`requestedSecretName` come from the caller's current request, and
`masterKeyId` / `dekVersion` come from the row. Copying the whole row
into a different workspace does not grant decryption because the AAD
reconstructed from the destination context does not match the AAD the
original ciphertext was authenticated under. A row in workspace A's
backup, inserted into workspace B's secret collection, fails AEAD
authentication at first decrypt and the subscription auto-pauses with
`secretsUnreadable`.

The row's `space` is still a `Ref<Space>` because `TDoc` requires one;
space-security still restricts who can even query the collection, but
decryption no longer depends on any value the row controls.

### Encryption

- **Scheme:** AES-256-GCM with a 96-bit random nonce per encrypt. GCM
  gives confidentiality and integrity in one primitive and matches
  the stored auth tag.
- **Keys:**
  - **Master keys** are 32-byte values held by the service.
    Identified by opaque `masterKeyId` strings. The service holds a
    **key ring**: one `active` master key plus any number of
    retired/pending-retirement keys. Rotation adds a new `masterKeyId`
    to the ring and marks it `active`; existing rows keep their old
    `masterKeyId` until a background re-encrypt rolls them forward.
    A row encrypted under `masterKeyId: m1` stays readable as long as
    `m1` is in the ring, even after `m2` becomes `active`.
    See [Key succession](#key-succession) for the rotation runbook
    (fixing point #9).
  - **Workspace DEKs** are HKDF-derived per workspace from the row's
    master key: `HKDF(masterKey_m1, info = "webhooks-dek-v<N>|<workspaceUuid>")`.
    `dekVersion` identifies the derivation; a bump is a cheap way to
    retire a specific DEK without rotating the whole master key.
- **No plaintext in Tx bodies.** Secret writes go through a dedicated
  service RPC. The management RPC encrypts in-service and inserts
  via a service token. Client Tx that CUDs `TWebhookSecret` is
  refused by the permission middleware (see Authorization).
- **Service-only decryption.** The web client cannot decrypt and
  never sees master keys.

### Key succession (fixing point #9)

The service refuses to start without `WEBHOOKS_MASTER_KEY` set.
Rotation steps:

1. The self-hoster adds `WEBHOOKS_MASTER_KEY_NEXT=<new 32-byte value>`
   alongside the existing `WEBHOOKS_MASTER_KEY`, both in base64. The
   service recognizes both: `WEBHOOKS_MASTER_KEY` is in the ring as
   `m1` (`active`), the new one is loaded as `m2` (`pending`).
2. Admin triggers a rotation RPC. The service re-encrypts all rows
   from `m1` to `m2` as a background job, updating `masterKeyId`.
   The job pauses the subscription whose secrets it is actively
   re-encrypting (bounded; one subscription at a time).
3. When the sweep completes, the admin swaps the env vars:
   `WEBHOOKS_MASTER_KEY=<new value>`, drop `_NEXT`. The service
   restarts; `m2` becomes `active`, `m1` leaves the ring.
4. Backups taken during the rotation window have mixed `masterKeyId`
   values; restoring requires both keys to be in the ring for the
   restored period. The deployment docs call this out.

The implementation PR ships a `openssl rand -base64 32` snippet in
the deployment docs for self-hosters.

### Redaction (placeholder-first)

The service renders two parallel templates per attempt:

- **Request template:** real secret values substituted, context-escaped,
  HMACed, sent on the wire.
- **Diagnostic template:** `[SECRET:<name>]` substituted in place of
  each `{{ secrets.<name> }}` BEFORE any URL-component or JSON
  encoding. Stored in audit and shown in previews.

Rev 2's substring-match-on-rendered-bytes approach would have missed
URL-encoded forms (`a/b` → `a%2Fb`) and receiver echoes. Rev 3
switched to placeholder-first for the outgoing side; rev 4 extends
the same discipline to the INCOMING side: test / preview responses
are not stored or shown by default (see
[Send test event](#send-test-event)).

### No `lastFour`

A secret of four characters or fewer would be fully leaked by a
suffix. `secretsMeta` exposes name + `lastRotatedAt` only.

## Target

### Default: minimal body

A subscription with `useMinimalBody: true` and no `bodyTemplate`
sends the minimal JSON body with no template to write. Default when
the user picks the bare `generic` preset without customization.

### Minimal body

```json
{
  "event": "IssueStatusChange",
  "txId": "<Ref<Tx>>",
  "deliveryId": "<uuid>",
  "timestamp": 1793235782113,
  "timestampIso": "2026-10-31T06:00:00.000Z",
  "workspace": { "uuid": "<WorkspaceUuid>", "name": "Pixelique" },
  "project": {
    "id": "<Ref<Project>>",
    "identifier": "CAL",
    "name": "Content Calendar"
  },
  "issue": {
    "id": "<Ref<Issue>>",
    "identifier": "CAL-14",
    "title": "Green Friday",
    "url": "<workspace URL>/#tracker/<project>/<issue>"
  },
  "actor": {
    "accountUuid": "<AccountUuid>",
    "displayName": "Ahmad Ismail"
  },
  "previousStatus": {
    "id": "<Ref<IssueStatus>>", "name": "Design",
    "categoryRef": "<Ref<StatusCategory>>", "categoryName": "Active"
  },
  "newStatus": {
    "id": "<Ref<IssueStatus>>", "name": "Review",
    "categoryRef": "<Ref<StatusCategory>>", "categoryName": "Active"
  }
}
```

Issue title and `actor.displayName` are sensitive data; see
[Audit](#audit).

### Where `previousStatus` comes from

See [Durable outbox](#durable-outbox-fixing-point-2). The pre-image
is captured inside the same storage transaction as the source
mutation; a crash before capture produces no outbox row and no event.
`null` means "unknown" (schema migration, pre-rollout history), not
"unchanged".

### Templates and variables

Variable set (fixed, documented, filled under the owner's current
permissions):

| Variable | Value |
|---|---|
| `{{ event }}` | `IssueCreated` / `IssueUpdated` / `IssueStatusChange` |
| `{{ txId }}` | triggering Tx id |
| `{{ deliveryId }}` | delivery uuid |
| `{{ timestamp }}` | ms since epoch |
| `{{ timestampIso }}` | ISO-8601 UTC |
| `{{ workspace.uuid }}`, `{{ workspace.name }}` | workspace facts |
| `{{ project.id }}`, `{{ project.identifier }}`, `{{ project.name }}` | project facts |
| `{{ issue.id }}`, `{{ issue.identifier }}`, `{{ issue.title }}`, `{{ issue.url }}` | issue facts |
| `{{ actor.accountUuid }}`, `{{ actor.displayName }}` | the Tx author |
| `{{ previousStatus.id }}`, `{{ previousStatus.name }}`, `{{ previousStatus.categoryRef }}`, `{{ previousStatus.categoryName }}` | prior status |
| `{{ newStatus.id }}`, `{{ newStatus.name }}`, `{{ newStatus.categoryRef }}`, `{{ newStatus.categoryName }}` | new status |
| `{{ secrets.<name> }}` | named secret |

Rules (v1):

- Placeholders are literal. Missing variable → empty string +
  `templateMiss` warning.
- No conditionals, loops, expressions, nested placeholders.
- Context-aware escaping:
  - **URL template**: values URL-component-encoded; secrets URL-encoded.
  - **Header values**: values inserted raw; CR / LF / NUL in rendered
    value rejected as `headerInjection`.
  - **JSON body** (media type normalized per RFC 7231): placeholders
    only inside JSON string literals in the template; outside-string
    placeholder rejected at save. After render the body is parsed as
    JSON; parse failure → `templateError`.
  - **Other content types**: raw substitution.
- **URL scheme + host + port MUST be literal** in the template. Any
  `{{ ... }}` before the first `/` after scheme is rejected at save.
- Media type parsing: `application/json; charset=utf-8` normalizes to
  `application/json` for selecting structural substitution; parameters
  preserved on the wire.

### Binding (fixing point #10)

`target.configDigest` is a sha256 over the APPROVED CONFIG (the
templates, method, content type, URL template, header names and
value templates, body template, sorted list of referenced secret
names). It is NOT over rendered values.

Delivery compares the job's recorded `configDigest` against the
subscription's current `configDigest` on every attempt; a mismatch
is `bindingDrift` and no send happens. `{{ issue.title }}`
substituting different values does not break binding, because binding
hashes the TEMPLATE, not the RENDERED output.

Rendered destination constraints (SSRF, host literal, header
validity) are separate per-attempt validations that run before send;
they fail on bad renders without implicating binding.

### Reserved and generated headers (fixing point #16)

Reserved header names (case-insensitive) rejected at save:

- `Host`, `Content-Length`, `Transfer-Encoding`, `Connection`,
  `Keep-Alive`, `Proxy-*`, `Trailer`, `TE`, `Upgrade`
- Any header name starting with `X-Huly-`
- **Generated names**: `Content-Type` and `User-Agent` (new in rev
  4; rev 3 let the user set `Content-Type` and the service appended
  its own, which allowed a mismatched header pair). The user picks
  a content type via the `target.contentType` field; the service
  owns the `Content-Type` header value exactly once.

Also rejected: duplicate header names at save; duplicate rendered
header names at render; empty-value rendered headers (dropped, with
`omittedHeaders` recorded in audit).

Full final header set (user + generated) is validated for uniqueness
and valid names/values before send.

### Service-generated headers

The service always emits exactly these, in addition to validated user
headers:

- `Content-Type: <normalized content type>`
- `User-Agent: huly-webhooks/<serviceVersion>`
- `X-Huly-Event: <event>`
- `X-Huly-Delivery: <deliveryId>`
- `X-Huly-Subscription: <subscriptionId>`
- `X-Huly-Replay-Window: <replayHorizonSeconds>` — advertises the
  receiver's required dedup retention window; a receiver that holds
  `X-Huly-Delivery` for at least this many seconds rejects exact
  replays. Fixes point #14 (receiver dedup window advertised).
- One or more `X-Huly-Signature` headers (grammar below).

## Signatures (fixing points #15, #14)

### Grammar

Formal grammar for `X-Huly-Signature`:

```
signature-header   = signature-element *( "," OWS signature-element )
signature-element  = version SP ";" param *( ";" param )
version            = "v=" DIGIT
param              = key "=" value
key                = "t" / "id" / "s" / "kid"
value              = <ASCII, no CR/LF/,/;>
```

Required params for `v=2`:

- `v=2`
- `t=<unix-ms timestamp of this attempt>`
- `id=<deliveryId>`
- `s=<lowercase-hex HMAC-SHA256>`
- `kid=<signing key id>` (new in rev 4)

Signed string:

```
"v=2;t=<ms>;id=<deliveryId>;kid=<keyId>." + <bodyBytes>
```

The signed string explicitly covers `kid` so a receiver can look up
the right verifier key.

Receivers MUST accept both the repeated-header form (one
`X-Huly-Signature` per element) and the combined-header form
(elements comma-separated in one header). Both parse to the same
list. The service SHOULD emit the repeated-header form; some proxies
collapse to the combined form so receivers must handle it.

### Signing key per subscription

- Per-subscription random 32-byte key (CSPRNG). Not shared across
  subscriptions.
- Receivers obtain their verifier key out-of-band at subscription
  creation (shown once in the UI, same reveal-and-store flow as
  the first secret rotation).
- Rotation creates a new signing key with a new `kid` and marks the
  old one `pendingRetirement` for `signingKeyGracePeriod` (default
  600 s). During the grace window the service emits BOTH signatures
  under their respective `kid` values, so a receiver running old +
  new verifier keys does not drop deliveries.

### Replay horizon

- Service advertises `X-Huly-Replay-Window: <seconds>`. Receivers
  that persist `X-Huly-Delivery` for at least this window reject
  exact replays within it.
- Default 300 s; subscription-level override (`replayHorizonSeconds`)
  for sensitive receivers can only LOWER the horizon (service-wide
  ceiling bounds the setting).

### Payload immutability per delivery (fixing point #14)

- A `deliveryId` is minted when the outbox row is created, not when
  the attempt runs.
- The outbox row stores the FINAL RENDERED REQUEST BYTES (URL,
  headers, body) alongside the deliveryId. All subsequent attempts
  for that deliveryId re-use those exact bytes. A re-render on a
  retry is forbidden.
- Only the signature header is re-generated per attempt (fresh `t`,
  same `kid`, same `s` computation over the frozen bytes). The
  receiver sees a stable payload under a stable deliveryId;
  signature freshness is purely for replay protection.
- A worker crash between a successful 2xx and the state write is
  retried with the same frozen payload; the receiver's dedup
  (within `X-Huly-Replay-Window`) absorbs the duplicate.

The rendered-request store is a service-local blob in the audit
blob store (same AEAD scheme as secrets), scoped by workspace +
`deliveryId`. See [Audit](#audit) for retention.

## SSRF protection

Checks run on the **final rendered URL** (post-templating). Scheme +
host + port must be literal in the template, so event data cannot
steer the destination.

### Address and transport policy

- **Public-only by default.** Reject if any resolved address falls in
  a non-public range. Explicit tables for IPv4 (including `127.0.0.0/8`,
  `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`,
  `100.64.0.0/10` CGNAT, `0.0.0.0/8`, `224.0.0.0/4` broadcast,
  `240.0.0.0/4` reserved, `192.0.0.0/24`, `192.0.2.0/24`,
  `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`) and IPv6
  (`::1/128`, `fc00::/7`, `fe80::/10`, `::/128`, IPv4-mapped
  `::ffff:0:0/96` with embedded IPv4 re-checked against the IPv4
  table, IPv4-compatible `::0.0.0.0/96`, 6to4 `2002::/16`,
  Teredo `2001::/32`).
- **IPv4-literal normalization.** `127.1`, `0x7f.0.0.1`, octal
  forms, and any other `net.isIP` representation are normalized
  before the range check.
- **Validate every A/AAAA.** The full RRset is validated per attempt;
  mixed public + private is rejected.
- **Connection pinning.** The HTTP client connects to the explicitly
  validated IP via a custom `lookup` function. TLS hostname
  verification stays against the subscription URL hostname.
- **Explicit proxy (fixing point #7).** When `WEBHOOK_EXPLICIT_PROXY`
  is set, the service does NOT delegate hostname resolution to the
  proxy. Instead, the service resolves the origin itself under the
  SSRF rules, then issues an HTTP CONNECT to the proxy targeting
  `<validated-origin-ip>:<port>` (not the hostname). This stops a
  proxy from resolving the hostname independently and connecting to
  a private address. TLS verification happens end-to-end against
  the origin hostname across the CONNECT tunnel. The proxy host
  itself is validated under the same SSRF rules at service start.
  Internal exceptions:
  - `WEBHOOK_ALLOWED_HOSTS` is a list of
    `host:port[,host:port...]` entries, matched exactly, bypassing
    the non-public range check for that host+port pair.
  - `WEBHOOK_ALLOWED_CIDRS` is a list of CIDR ranges + required
    port ranges, bypassing the non-public check for destinations
    inside those ranges. New in rev 4 because
    `WEBHOOK_ALLOWED_HOSTS` cannot cover a self-hoster who allows
    an entire internal subnet (e.g. a Docker overlay network).
- **No implicit proxy.** `HTTP_PROXY` / `HTTPS_PROXY` /
  `NO_PROXY` / system proxy / PAC are ignored. Only
  `WEBHOOK_EXPLICIT_PROXY` opts into a proxy.
- **Redirects disabled.** 3xx is a delivery error (`errorReason:
  'redirect'`).
- **HTTPS by default.** `http://` only when
  `WEBHOOK_ALLOW_HTTP=true` AND the host is in the allowlist.
- Timeout default 10 s, hard cap 30 s. Response body cap 8 KB.

## Durable outbox (fixing point #2)

Rev 3 proposed writing the outbox row in the same Tx as the apply
and relied on cross-domain atomicity. Michael pointed out that
Postgres processes domains separately in the existing
`foundations/server/packages/postgres/src/storage.ts` so a cross-
domain batch is not actually atomic.

Rev 4 uses a **journal-based recovery** design:

### Journal

A single `TWebhookJournal` entry is written AT THE END of the Tx
middleware pipeline for every eligible source Tx:

```ts
TWebhookJournal extends TDoc {
  space: Ref<Space>
  // State machine. 'pending' = captured, outbox rows not yet written.
  // 'realized' = all outbox rows written. 'failed' = outbox write
  // failed terminally.
  state: 'pending' | 'realized' | 'failed'
  // Full capture of what the outbox needs to realize.
  sourceTx: {
    txId: Ref<Tx>
    event: 'IssueCreated' | 'IssueUpdated' | 'IssueStatusChange'
    issueRef: Ref<Issue>
    projectRef: Ref<Project>
    actorPersonId: PersonId
    previousStatusRef: Ref<IssueStatus> | null
    newStatusRef: Ref<IssueStatus> | null
    txModifiedOn: Timestamp
  }
  // Matching subscriptions resolved at capture time.
  matchedSubscriptions: Array<{ subscriptionId: Ref<WebhookSubscription>, revision: number }>
  attemptCount: number
  lastAttemptAt: Timestamp | null
  createdAt: Timestamp
}
```

The journal is in the **same domain** as the subscription docs, so a
single-domain storage transaction suffices for its write. The pre-
image (`previousStatusRef`) is captured by reading the issue state
INSIDE the Tx middleware before `provideTx` is called, matching the
same slot where a trigger would read it, but WITHOUT relying on a
cross-domain commit boundary.

### Realization

A worker sweeps `state: 'pending'` journal entries and realizes them
into outbox rows:

```ts
TWebhookOutboxJob extends TDoc {
  space: Ref<Space>
  subscription: Ref<WebhookSubscription>
  revision: number                 // frozen at realize time
  event: WebhookTrigger['event']
  deliveryId: string               // uuid; minted at realize time
  journal: Ref<TWebhookJournal>
  txId: Ref<Tx>
  frozenRequest: {
    method: 'POST' | 'PUT'
    url: string                    // final rendered
    headers: Array<{ name: string, value: string }>
    bodyBytes: Buffer
    bodyBytesHash: string          // sha256; used for audit integrity
    configDigestAtRealize: string  // for bindingDrift re-check
    renderedAt: Timestamp
  }
  state: 'pending' | 'in_flight' | 'done' | 'failed' | 'quarantined'
  attempts: number
  nextAttemptAt: Timestamp | null
  terminalReason: string | null
  // Set when a worker claims the subscription-wide lease (not this row).
  // See "Ordering".
  claimedUnderLease: string | null
}
```

Realization steps for one journal entry:

1. Mark journal `in_realization` with an optimistic lock.
2. For each matched subscription, resolve the owner's current source
   access. If revoked, write a `denied` audit row directly (no
   outbox row) and move on.
3. Compute the final rendered request bytes (URL / headers / body)
   under the current secret values. Store those in `frozenRequest`.
4. Insert the outbox row.
5. When all outbox rows for the journal entry are inserted, mark the
   journal `realized`.

A crash during realization leaves the journal `pending` /
`in_realization`; the sweeper retries after lease expiry. Realization
is idempotent: a journal entry with partial outbox rows will complete
on the next sweep (unique index on `(journal, subscription)` prevents
duplicate outbox rows for the same realization).

A journal entry that fails realization repeatedly (default 5
attempts over bounded time) is marked `failed` and listed in the
quarantine UI; it does not block other journal entries.

### Dedup key

`(subscriptionId, event, txId)` is the full dedup key. A single Tx
firing both `IssueUpdated` and `IssueStatusChange` produces TWO
outbox rows (distinct `event`), matching rev 3.

### Kafka role (fixing point #11)

Kafka is the **wake-up signal**, not the source of truth. The
delivery service subscribes to the Tx stream to notice journal
entries promptly; the sweeper is the authoritative worker and runs
on an interval regardless of Kafka. **Kafka ack happens after the
journal write succeeds** — NOT after HTTP terminal — so a slow
receiver on one subscription cannot stall unrelated workspaces
sharing a partition.

Subscription-key routing is NOT done through Kafka. Rev 3 implied
Kafka could route per `(workspaceUuid, subscriptionId)`; it cannot
on the existing topic. Rev 4 does subscription routing in the outbox
layer via a subscription-wide lease (see Ordering).

## Ordering, claiming, backlog (fixing points #12, #13)

### Subscription-wide lease

The delivery service claims **a subscription**, not a single row.
When a worker takes `TWebhookSubscriptionLease` for subscription S,
it is the only worker that may process outbox rows for S until the
lease expires. The lease has a short TTL (default 30 s) and the
worker renews it while actively processing; crash + no renewal =
lease expires and another worker claims.

Within a lease the worker processes outbox rows for S in strict
`nextAttemptAt` ascending order. A newer `pending` job in S cannot
overtake an older job that is between retries unless the older job
is `quarantined` (which releases the slot explicitly). Fixes point
#13: no row-level race, no retry overtaking without quarantine.

### Backlog quotas (fixing point #12)

- **Per subscription backlog cap:** `maxOutboxPerSubscription`
  default 10 000 rows. When exceeded, new realizations for that
  subscription write a `denied` audit row with
  `errorReason: 'backlogFull'` and the subscription auto-pauses.
- **Per workspace backlog cap:** `maxOutboxPerWorkspace` default
  100 000 rows across all subscriptions. Enforced the same way.
- **Age cap:** an outbox row older than
  `maxOutboxAgeSeconds` default 7 days is marked `quarantined`;
  the admin triages.
- **Fan-out cap:** a single Tx that matches more than
  `maxFanoutPerTx` default 50 subscriptions marks the journal
  `failed:fanoutExceeded`. Prevents a misconfiguration (many
  subscriptions with `allProjects + no filter`) from exploding a
  single Tx into unbounded work.
- **Retry budget:** retries + test events + initial attempts all
  draw from the same per-subscription + per-workspace sliding-window
  budgets.
- **Quarantine retention:** `quarantinedJob` rows are kept for
  `quarantineRetentionDays` default 7, then auto-deleted; the audit
  rows for their attempts stay on the normal audit schedule. UI
  shows them with "triage before auto-delete" countdown.

### Overflow

When any quota triggers:

- The subscription is auto-paused.
- A UI notice names the quota that fired.
- The source Tx apply is unaffected (never blocks on webhook
  delivery backlog).
- The service rejects new management writes to the subscription
  until the Owner triages (prevents a dashboard-driven loop re-arming
  a saturated subscription).

## Audit (fixing point #4)

Audit payloads are **sensitive data by definition** (they carry
rendered request URLs, headers and bodies; they reference issue
titles and actor display names). Rev 3 kept them in ordinary Doc
collections; querying `TWebhookDeliveryAttempt` would get space-
filtered on `objectSpace` which does NOT reach the referenced
issue's space. So a user who lost source access could still read
audit rows that leaked the issue content.

Rev 4 moves sensitive audit content **out of ordinary Tx history**:

### Audit blob store

```ts
TWebhookAuditBlob extends TDoc {
  space: Ref<Space>              // workspace settings space
  subscription: Ref<WebhookSubscription>
  deliveryId: string
  attempt: number
  blobId: string                 // reference into the dedicated store
  masterKeyId: string
  dekVersion: number
  ciphertext: Buffer             // audit JSON encrypted with same AEAD as secrets
  nonce: Buffer
  tag: Buffer
  at: Timestamp
}
```

The audit JSON holds the rendered URL (diagnostic form with
`[SECRET:*]`), header names + diagnostic values, body (if the
subscription opts in via `storeResponseBody`), HTTP code, response
headers (if opted in), response body (if opted in, first 2 KB),
`templateWarnings`, and the error reason.

The blob is written in the subscription's workspace settings space,
same as the subscription doc, so space-security normally applies. On
every read, the service additionally verifies the **current reader
has source access to the referenced issue**, using the issue ref
captured in the parent `TWebhookDeliveryAttempt` row. A reader without
source access gets back a `redacted` row that only shows outcome +
error reason + timestamp — no identifiers, no title, no displayed
name.

Audit **client broadcasts are suppressed**: the audit blob store is
flagged as a non-broadcast domain so a workspace with 100 Owners
does not fan out every delivery event. Clients query the audit UI
through a dedicated RPC that re-authorizes on each read.

### Attempt metadata row

```ts
TWebhookDeliveryAttempt extends TDoc {
  space: Ref<Space>
  subscription: Ref<WebhookSubscription>
  deliveryId: string
  attempt: number
  event: WebhookTrigger['event'] | 'test' | 'skipped' | 'rotated' | 'bindingChanged'
  txId: Ref<Tx>
  issueRef: Ref<Issue>           // for current-source-access gating
  at: Timestamp
  outcome: 'ok' | 'error' | 'skipped' | 'denied'
  httpCode?: number
  errorReason?: string
  blobId?: string                // reference to TWebhookAuditBlob when outcome allows one
  omittedHeaders?: string[]
}
```

The row carries identifiers + outcome + error reason. The sensitive
rendered-content lives in `TWebhookAuditBlob` and is read-gated.

### Retention per layer

- `TWebhookDeliveryAttempt` rows: `attemptRetentionDays` default
  30.
- `TWebhookAuditBlob`: same retention as its paired attempt row;
  cleanup worker removes blob + attempt + Tx history as one unit
  (see Cleanup).
- `TWebhookOutboxJob`: `done` or `failed` kept for
  `outboxRetentionDays` default 7.
- `TWebhookJournal`: `realized` kept for `journalRetentionDays`
  default 7; `pending` and `in_realization` are kept until they
  reach a terminal state regardless of age.

### Cleanup worker (fixing point #17)

The worker records a **durable cleanup intent** before any delete.

```ts
TWebhookCleanupTask extends TDoc {
  space: Ref<Space>
  targets: Array<{ collection: string, docId: Ref<Doc> }>
  state: 'pending' | 'documentsRemoved' | 'historyPruned' | 'done'
  attempts: number
  lastError?: string
  createdAt: Timestamp
}
```

Flow per sweep:

1. Build a batch of expired docs across the four collections
   (`maxCleanupBatch` default 500).
2. Write a `TWebhookCleanupTask` with the targets, state `pending`.
3. Issue `TxRemoveDoc` for each target. On success, mark state
   `documentsRemoved`.
4. Issue the low-level history-pruning helper (same shape as
   existing audit roll-offs; the implementation PR will reference
   the exact helper by name). On success, mark state
   `historyPruned`, then `done`.
5. On failure at any step, retain the task row; the next sweep
   retries idempotently.
6. **Orphan reconciliation**: if a cleanup task is `pending` for
   more than `cleanupStuckThresholdDays` default 7, write a
   `cleanupStuck` metric and page the operator. The row stays in
   `pending` until human triage; the worker never auto-abandons.

Confidential history is NOT indefinitely retained if a step partially
succeeds, because the cleanup task row keeps the pending work
visible to the next sweep.

### Backups and client broadcasts

- The audit blob domain is excluded from the ordinary client
  broadcast list; a Tx that writes a `TWebhookAuditBlob` does not
  push through `broadcast.txes`. Clients read via RPC which
  re-authorizes.
- Backups keep original data as before. Restore handling (next
  section) pauses all subscriptions on restore so the restored
  audit history is never dispatched or broadcast until an Owner
  explicitly reactivates.

## Restore and clone (fixing point #8)

Rev 3 relied on cross-workspace decryption failure to pause
subscriptions on restore. A same-workspace restore that keeps the
UUID and the master key decrypts fine and would resume production
sends. Rev 4 pauses **unconditionally** on every restore:

- The restore path (`server/backup/src/restore.ts`) sets a workspace-
  level flag `webhooksRestoreGuardSet=true` as its last step.
- On service start AND on every subscription read, the delivery
  service checks the flag. When set, all subscriptions in the
  workspace are forced to behave as `enabled: false` for delivery
  purposes, regardless of the stored field.
- Any `pending` outbox rows that existed before restore are
  quarantined with `errorReason: 'restoreGuard'` so they do not
  fire later.
- A UI banner shows "Subscriptions paused after restore. Review
  destinations and credentials before reactivating."
- The flag clears only when an Owner opens each subscription,
  confirms destination + credentials, and runs an explicit
  reactivation action. The reactivation is per-subscription so a
  workspace with many subscriptions does not have to review all at
  once; the banner shows a count of still-paused ones.
- Clone operations (if any) set the same flag.

## UI

Workspace settings > Webhooks tab. Owner-only. Hidden entirely when
`PluginConfiguration.enabled === false`.

Per subscription row:

- Name, preset badge, URL host (full URL hidden; secrets masked).
- Enabled toggle (atomic apply).
- Last delivery: timestamp, status, HTTP code, diagnostic body
  preview with `[SECRET:*]` (only when `storeResponseBody` is on
  and the reader has source access to the referenced issue).
- Actions: Edit (stored owner only), Send test event (stored owner
  only), Delete (current Owner).

Editor:

- Name
- Preset selector (apprise / slack / discord / generic)
- Method, URL template (host literal; URL preview on blur)
- Header list (reserved + generated names blocked; CR / LF rejected
  on blur)
- Content type + body template OR "Use minimal body"
- Named secrets panel: add / rotate / remove (service RPC;
  write-only; `lastRotatedAt` shown).
- Triggers: event + `projectScope` + `statusFilter.on`.
- Retry policy (collapsed).
- **Render preview**: picks a recent matching issue the caller can
  read; renders the diagnostic URL / headers / body with
  `[SECRET:*]`; shows what will be POSTed. Preview requires caller +
  stored owner source-access.

### Send test event (fixing point #5)

- Picks a recent matching Issue the caller + stored owner can both
  read.
- Builds the payload with real values and sends with the live
  signed body.
- **Response display defaults to status + headers metadata only**.
  Body is NOT shown. A collapsed section says "Receiver returned N
  bytes; show response body [requires acknowledgement]."
- Clicking "Show response body" presents an explicit disclosure
  warning: "The receiver may echo values of your stored
  credentials in its response. Showing the body may reveal
  credential bytes to you if the receiver echoed them.
  Continue?" Only on the second confirmation does the modal fetch
  and show the body.
- The response body is not persisted unless
  `storeResponseBody: true` on the subscription AND the caller
  acknowledges disclosure at save time.
- An audit row with `event: 'test'` is written in both cases;
  only the metadata row is persisted on default-no-body tests.

### Rotation UX

- Rotating a secret: service RPC regenerates; the UI shows the new
  value once (reveal-and-store flow); audit row `rotated` written.
  Externally-provisioned credentials (where the receiver issues the
  new value out-of-band) are a two-step: user pastes new value into
  the editor, then clicks Rotate; the UI calls that out as "Paste
  the new value from <receiver>, then rotate." Fixes a rev-3 nit
  about distinguishing external vs internal rotation.
- Rotating the subscription's signing key: separate action in the
  editor; shows the new verifier value once; the grace window is
  automatic (service emits both signatures until the window
  expires).

## Apprise and other presets

### `apprise` (fixing point #18 and rev-3 open question)

```yaml
preset: apprise
method: POST
# User types the Apprise base URL literally; validated as a host.
# WEBHOOK_ALLOWED_HOSTS / WEBHOOK_ALLOWED_CIDRS covers internal
# Apprise relays.
urlTemplate: "https://apprise.example/notify"
headerTemplates:
  # Optional. The editor drops this template entirely when
  # `apprise_bearer` has no stored value, BEFORE the save-time
  # validation that would otherwise fail on an unresolvable
  # secret reference.
  - name: Authorization
    valueTemplate: "Bearer {{ secrets.apprise_bearer }}"
    requiresSecret: 'apprise_bearer'    # new field: ties this header to a secret
contentType: application/json
useMinimalBody: false
bodyTemplate: |
  {
    "urls": "{{ secrets.apprise_urls }}",
    "title": "{{ issue.identifier }} {{ newStatus.name }}",
    "body": "{{ issue.title }} by {{ actor.displayName }}",
    "type": "info",
    "format": "markdown"
  }
```

A header template may declare `requiresSecret: '<name>'`. The save
pipeline **drops the whole header** when the named secret has no
value, THEN runs the "every referenced secret is present"
validation. An Apprise install without a bearer saves cleanly; the
rendered request simply omits `Authorization`.

### `slack` and `discord`

Credentials split into separate path segments so the structural `/`
is not URL-encoded. Three Slack components or two Discord
components; each is its own named secret.

### Preset validation parity

All presets go through the same validation as custom targets. No
fast path. A preset that uses an optional-with-`requiresSecret`
header passes exactly when the equivalent custom target would.

## Data model summary

```ts
// User-visible subscription doc.
TWebhookSubscription extends TDoc { ... }

// Journal: pending capture of an eligible source Tx.
// Lives in the same domain as subscriptions for single-domain
// atomicity.
TWebhookJournal extends TDoc { ... }

// Outbox row per (subscription, event, txId); authoritative queue.
TWebhookOutboxJob extends TDoc { ... }

// Attempt metadata row: identifiers, outcome, error reason, blob ref.
TWebhookDeliveryAttempt extends TDoc { ... }

// Encrypted blob holding rendered-request + rendered-response audit
// payloads. Non-broadcast; AEAD-encrypted; AAD reconstructed from
// authenticated context.
TWebhookAuditBlob extends TDoc { ... }

// Named secrets. Service-write only; AEAD-encrypted; AAD
// reconstructed from authenticated context.
TWebhookSecret extends TDoc { ... }

// Subscription-wide lease for ordered delivery.
TWebhookSubscriptionLease extends TDoc { ... }

// Durable cleanup intent; retains confidential history pruning
// state across worker crashes.
TWebhookCleanupTask extends TDoc { ... }
```

All eight are workspace-scoped; all eight route by the service's
authenticated `WorkspaceUuid`, never by doc fields.

## Testing

Unit:

- Permission middleware: non-Owner Tx creating / mutating a
  subscription is refused BEFORE storage write; compound Tx is
  walked recursively.
- Stored-owner-only destination change: Owner B's attempt to
  change A's subscription destination is refused with
  `ownerIneligible`.
- AEAD: AAD reconstructed from authenticated context; a row
  copied into another workspace fails decrypt.
- Master-key succession: rotation ring reads old + new rows
  simultaneously; retiring a key makes its rows unreadable.
- Journal realization: pre-image captured in-pipeline; crash
  before realize leaves a `pending` journal entry; sweeper
  realizes idempotently.
- Payload immutability: a retry uses the same bodyBytes as the
  first attempt; signature changes, body does not.
- Subscription-wide lease: two sweepers racing one subscription
  take exclusive leases; a newer job cannot overtake an older
  retry-pending job.
- Backlog quotas trip auto-pause and write a `backlogFull` audit
  denial.
- Signature grammar: repeated-header form and combined-header form
  parse to the same list; receiver sample code validates both; test
  vectors.
- SSRF: all private ranges on IPv4 + IPv6; IPv4-literal normalization;
  mixed public + private rejected; connection pinning; proxy CONNECT
  targets the validated IP; proxy host validated at start.
- Audit read gating: a reader without source access gets the
  redacted row shape.
- Test event body-disclosure: default returns no body; opt-in
  triggers explicit confirmation audit event.
- Cleanup task durability: TxRemoveDoc succeeds + history prune
  fails → task row retained; next sweep completes it.
- Apprise optional Authorization: absent `apprise_bearer` drops the
  header before validation; present value renders normally.

Integration:

- Full round-trip against a local HTTP sink: subscribe, trigger a
  Tx, assert signed body, assert audit blob encrypted with AEAD,
  assert attempt row carries identifiers.
- Restore handling: a same-workspace backup restored sets the
  guard flag; subscriptions behave as disabled until Owner
  reactivates each.
- Kafka ack ordering: a stalled HTTP receiver on sub A does not
  stall deliveries for sub B in another partition group.
- Cross-workspace copy of a secret row fails to decrypt in the
  destination workspace; subscription auto-pauses with
  `secretsUnreadable`.
- Owner demotion mid-queue: outbox job is refused on the next
  attempt with `ownerDemoted`; audit row written.

## Open questions for review

None blocking. If the implementation PR surfaces any, they go
through the usual review loop.
