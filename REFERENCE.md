# Proof Advanced Reference

This document covers advanced API details for Proof and Proof SDK deployments. For core operations (create, read, edit, comment, suggest), see [proof.SKILL.md](./proof.SKILL.md).

All examples use `$BASE_URL` — set it once:

```bash
BASE_URL="https://proofeditor.ai"
```

---

## Route Alias Table

Proof exposes multiple route aliases for compatibility. **Use canonical routes.**

| Category | Canonical | Compatibility | Legacy (avoid) |
|---|---|---|---|
| Create | `POST /documents` | `POST /share/markdown` | `POST /api/documents` |
| State | `GET /documents/:slug/state` | — | `GET /api/documents/:slug/open-context`, `GET /api/documents/:slug/info` |
| Snapshot | `GET /documents/:slug/snapshot` | — | — |
| Ops | `POST /documents/:slug/ops` | — | — |
| Edit | `POST /documents/:slug/edit` | — | — |
| Edit V2 | `POST /documents/:slug/edit/v2` | — | — |
| Presence | `POST /documents/:slug/presence` | — | — |
| Events | `GET /documents/:slug/events/pending` | — | — |
| Events Ack | `POST /documents/:slug/events/ack` | — | — |
| Collab Session | — | — | `GET /api/documents/:slug/collab-session` |

**Warning:** `/api/documents` and `/api/agent/*` routes are internal/legacy. They may be warned or disabled on hosted environments. Use canonical routes instead.

## Bridge Routes

Bridge-compatible deployments expose neutral bridge routes alongside canonical ones:

```text
GET  /documents/:slug/bridge/state
GET  /documents/:slug/bridge/marks
POST /documents/:slug/bridge/comments
POST /documents/:slug/bridge/suggestions
POST /documents/:slug/bridge/rewrite
POST /documents/:slug/bridge/presence
```

These mirror the canonical endpoints with bridge-neutral payloads. Use canonical routes unless integrating with a bridge-specific consumer.

## Structured Edit Operations (`/edit`)

For string-based edits without block refs. Prefer `/edit/v2` for most tasks.

```text
POST /documents/:slug/edit
```

All requests require `Content-Type: application/json`, auth via Bearer token, and a `by` field. The `operations` array supports up to 50 ops.

### Append to a section

Add content at the end of a named section (matched by heading text):

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/edit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "by": "ai:your-agent",
    "operations": [
      {"op": "append", "section": "Notes", "content": "\n\nNew content here."}
    ]
  }'
```

The `section` value matches heading text (e.g., `"Notes"` matches `### Notes`).

### Replace text

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/edit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "by": "ai:your-agent",
    "operations": [
      {"op": "replace", "search": "old text to find", "content": "new replacement text"}
    ]
  }'
```

### Insert after text

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/edit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "by": "ai:your-agent",
    "operations": [
      {"op": "insert", "after": "anchor text", "content": "\n\nInserted content."}
    ]
  }'
```

`insert` only supports `after`. Payloads using `before` are rejected with `INVALID_OPERATIONS`.

### Multiple operations

Operations are applied in order:

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/edit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "by": "ai:your-agent",
    "operations": [
      {"op": "append", "section": "Dan", "content": "\n\nNew idea."},
      {"op": "replace", "search": "(placeholder)", "content": "Actual content."}
    ]
  }'
```

### Optimistic locking

Pass `baseUpdatedAt` (from a prior state response) to detect concurrent edits:

```json
{"by": "ai:your-agent", "baseUpdatedAt": "2026-02-16T...", "operations": [...]}
```

If the document's `updatedAt` doesn't match, you'll get a `409` with `retryWithState`.

### Response

```json
{
  "success": true,
  "slug": "<slug>",
  "updatedAt": "<ISO timestamp>",
  "collabApplied": true
}
```

- `collabApplied: true` — edit was pushed into the live collab session.
- `presenceApplied` — only `true` when you supplied `X-Agent-Id`, `agentId`, or `agent.id`.

## Edit V2 Deep Dive

### Convergence fields

After a successful `/edit/v2`, the response includes collab convergence status:

- `collab.status` — compatibility status (`confirmed|pending`), fragment-authoritative.
- `collab.fragmentStatus` — ProseMirror/Yjs fragment convergence (`confirmed|pending`).
- `collab.markdownStatus` — SQL markdown projection convergence (`confirmed|pending`).
- `collabApplied` follows `fragmentStatus` (not markdown projection status).
- `202` is only returned when fragment convergence is pending.

### Precondition contract

- `baseRevision` is **required** on `/edit/v2`.
- `baseUpdatedAt` is **not accepted** on `/edit/v2` (use `/edit` instead for timestamp-based locking).

### Idempotency

- Send `Idempotency-Key` header (`X-Idempotency-Key` also accepted for compatibility).
- Block-level retries are common in automation — always include this header.
- `IDEMPOTENCY_KEY_REUSED`: same key reused with a different payload hash.

## Mutation Contract Discovery

Read `contract` from `GET /documents/:slug/state` to detect the current mutation requirements:

| Field | Values | Meaning |
|---|---|---|
| `contract.mutationStage` | `A`, `B`, `C` | Rollout stage for mutation requirements |
| `contract.idempotencyRequired` | `true/false` | Whether `Idempotency-Key` is mandatory |
| `contract.preconditionMode` | varies | Current precondition requirements |

### Stage progression

- **Stage A**: Idempotency key optional, preconditions optional.
- **Stage B**: Idempotency key recommended, preconditions recommended.
- **Stage C**: Idempotency key required, preconditions required.

### Mutation error codes

| Code | Meaning | Retryable |
|---|---|---|
| `IDEMPOTENCY_KEY_REQUIRED` | Mutation omitted key in required stage | Yes (add header) |
| `IDEMPOTENCY_KEY_REUSED` | Same key, different payload hash | No (generate new key) |
| `BASE_REVISION_REQUIRED` | Stage requires `baseRevision` | Yes (fetch snapshot) |
| `LIVE_CLIENTS_PRESENT` | Rewrite blocked by active collab clients | Yes (wait or use edit/v2) |
| `REWRITE_BARRIER_FAILED` | Rewrite safety barrier failed | Yes (retry with backoff) |

For `LIVE_CLIENTS_PRESENT`: use `retryWithState` to refresh, confirm `connectedClients === 0`. If `forceIgnored=true`, do not retry with `force` in hosted environments.

## Title Metadata

```bash
curl -sS -X PUT "$BASE_URL/documents/$SLUG/title" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Updated document title"}'
```

Discovery: `GET /documents/:slug/state` includes `_links.title` and `agent.titleApi`.

## Collab Session Lifecycle

For WebSocket-based real-time collaboration:

1. Resolve open context and capabilities via `GET /documents/:slug/state`.
2. Join collab using `session.collabWsUrl` and `session.token`.
3. Refresh before token expiry using the collab-refresh endpoint from `session._links` (note: this route currently uses the `/api/documents/:slug/collab-refresh` legacy path — no canonical alias exists yet).
4. Reconnect using the refreshed token.

## Presence and Event Polling (Full Details)

### Polling

```text
GET /documents/:slug/events/pending?after=<cursor>&limit=100
```

The `after` parameter is a cursor (event ID). Start with `0` for initial fetch.

### Acknowledging

```text
POST /documents/:slug/events/ack
Body: {"upToId": <cursor>, "by": "ai:your-agent"}
```

### Presence

Send `X-Agent-Id: <your-agent-id>` on `GET /state` to register presence passively, or send explicit presence:

```text
POST /documents/:slug/presence
Body: {"agentId":"your-agent","status":"thinking","summary":"Reviewing section 2"}
```

## Environment Auth Modes

### Direct-share auth (`PROOF_SHARE_MARKDOWN_AUTH_MODE`)

| Mode | Behavior |
|---|---|
| `none` | Open route — good for local/dev |
| `api_key` | Requires `PROOF_SHARE_MARKDOWN_API_KEY` as Bearer token on create |
| `auto` | Resolves to `none` by default in Proof SDK |

### Legacy create mode (`PROOF_LEGACY_CREATE_MODE`)

Controls `POST /api/documents`:

| Mode | Behavior |
|---|---|
| `allow` | Accepts requests normally |
| `warn` | Accepts but logs deprecation warning |
| `disabled` | Returns 410 Gone |
| `auto` | Default behavior for the deployment |

## Troubleshooting

### `ANCHOR_NOT_FOUND` on `/edit` replace or insert

The `/edit` endpoint searches for `search` or `after` text in the document. Agent-edited documents may contain internal `<span data-proof="authored">` HTML tags. The search automatically falls back to matching against clean text (tags stripped). If it still fails, the text genuinely doesn't exist — re-read state and verify your quote.

### `LIVE_CLIENTS_PRESENT` on `rewrite.apply`

`rewrite.apply` is blocked when authenticated collaborators are connected. On hosted environments, `force` is ignored. Alternatives:
1. Use `/edit` or `/edit/v2` instead (they work with live clients).
2. Wait for clients to disconnect (poll `/state`, check `connectedClients`).

### Suggestion anchors not matching

`suggestion.add` resolves quotes against clean text even when stored markdown contains internal annotations. If you still get `ANCHOR_NOT_FOUND`, re-read state and verify the quote text.

### Content drift after suggestion reject cycles

Repeated suggest/reject cycles now preserve stable suggestion anchors. If you see unexpected content drift, re-read via `Accept: text/markdown` and report the exact request/response pair.

### `COLLAB_SYNC_FAILED`

API edits can fail when a browser has the document open with an active Yjs collab session. `/edit` and `/edit/v2` handle this gracefully, but `rewrite.apply` does not. Retry after a short delay or use `/edit`/`/edit/v2` instead.
