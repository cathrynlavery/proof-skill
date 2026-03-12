---
name: proof
description: Work with hosted Proof documents and Proof SDK-compatible deployments over HTTP. Use when creating, reading, editing, or reviewing shared docs.
---

# Proof

Proof is the hosted collaborative document editor at [proofeditor.ai](https://proofeditor.ai). Proof SDK is the open-source editor, collaboration server, and agent bridge. Both use the same document model and HTTP patterns described here.

## Configuration

```bash
BASE_URL="https://proofeditor.ai"
```

Override for self-hosted Proof SDK deployments:

```bash
BASE_URL="https://your-proof-instance.example"
```

All examples below use `$BASE_URL`. Set it once and every command works unchanged.

## Quick Start: You Received a Proof Link

No browser needed. Use HTTP directly (`curl`, `web_fetch`, etc.).

Given a shared link like `https://proofeditor.ai/d/<slug>?token=<token>`:

1. Extract `slug` and `token` from the URL.
2. Fetch the document using content negotiation on the same URL:

```bash
# JSON (recommended) — includes markdown, _links, auth hints
curl -sS -H "Accept: application/json" "$BASE_URL/d/$SLUG?token=$TOKEN"

# Raw markdown
curl -sS -H "Accept: text/markdown" "$BASE_URL/d/$SLUG?token=$TOKEN"
```

The JSON response includes `markdown`, `_links` (state, ops, docs), and `agent.auth` hints.

## Quick Start: Create a Document

```bash
curl -sS -X POST "$BASE_URL/documents" \
  -H "Content-Type: application/json" \
  -d '{"title":"My Document","markdown":"# Hello\n\nFirst draft."}'
```

Hosted Proof also accepts `POST /share/markdown` as a compatibility alias.

### What to Persist After Create

The response includes fields you must save for follow-up operations:

| Field | Purpose |
|---|---|
| `slug` | Document identifier used in all API paths |
| `shareUrl` | Human-readable URL to share with users |
| `tokenUrl` | Share URL with token embedded (ready to hand out) |
| `ownerSecret` | Full-owner credential — store securely, never expose in UI |
| `accessToken` | Scoped link credential for viewer/commenter/editor operations |
| `_links` | Hypermedia links to state, ops, events, and docs endpoints |

## Core Rules

- Include `by` on **every** write. Use `ai:your-agent` (replace `your-agent` with your agent name).
- Send `Idempotency-Key` header on all mutation requests so retries are safe.
- Use only canonical routes (`/documents/...`). Avoid `/api/documents` and `/api/agent/*` — these are legacy/internal and may be disabled.

## Authentication & Tokens

### Passing the token

Use one of (in preference order):

1. `Authorization: Bearer <token>` (preferred)
2. `x-share-token: <token>`
3. `?token=<token>` query parameter

### ownerSecret vs accessToken

| Token | Scope | Use for |
|---|---|---|
| `ownerSecret` | Full owner | Pause, resume, revoke, delete, owner-level ops |
| `accessToken` | Scoped to link role | Read, comment, suggest, edit (per role) |

Use `accessToken` for non-owner operations. Store `ownerSecret` securely and never expose it in user-facing UI.

## Read Document State

### Full state

```bash
curl -sS "$BASE_URL/documents/$SLUG/state" \
  -H "Authorization: Bearer $TOKEN"
```

Include `X-Agent-Id: your-agent` only when you want presence to appear.

### Snapshot (for structured edits)

```bash
curl -sS "$BASE_URL/documents/$SLUG/snapshot" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `revision`, an ordered `blocks` array with stable refs (`b1`, `b2`, ...), and clean markdown per block.

### Content negotiation on share URLs

```bash
curl -sS -H "Accept: application/json" "$BASE_URL/d/$SLUG?token=$TOKEN"
curl -sS -H "Accept: text/markdown" "$BASE_URL/d/$SLUG?token=$TOKEN"
```

## Edit with Block Operations (Recommended)

The most reliable edit workflow: get a snapshot, then apply block-level edits.

### Step 1: Get snapshot

```bash
curl -sS "$BASE_URL/documents/$SLUG/snapshot" \
  -H "Authorization: Bearer $TOKEN"
```

Each block has `ref` (e.g., `b3`), `markdown`, and `type`.

### Step 2: Apply edits

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/edit/v2" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "by": "ai:your-agent",
    "baseRevision": 42,
    "operations": [
      {"op": "replace_block", "ref": "b3", "block": {"markdown": "Updated paragraph."}},
      {"op": "insert_after", "ref": "b3", "blocks": [{"markdown": "## New Section"}]}
    ]
  }'
```

On success: returns new `revision`, updated `snapshot`, and `collab` status.
On `STALE_REVISION`: response includes latest snapshot — re-read blocks and retry.

## Comments, Suggestions, and Rewrites

Use `POST /documents/<slug>/ops` for reviewable operations.

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/ops" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '<operation JSON below>'
```

### Available operations

| Type | Payload | Description |
|---|---|---|
| `comment.add` | `{"type":"comment.add","by":"ai:your-agent","quote":"anchor text","text":"Comment body"}` | Add a comment anchored to text |
| `comment.reply` | `{"type":"comment.reply","by":"ai:your-agent","commentId":"<id>","text":"Reply body"}` | Reply to an existing comment |
| `comment.resolve` | `{"type":"comment.resolve","by":"ai:your-agent","commentId":"<id>"}` | Resolve a comment thread |
| `suggestion.add` | `{"type":"suggestion.add","by":"ai:your-agent","kind":"replace","quote":"old text","content":"new text"}` | Suggest a text replacement |
| `suggestion.accept` | `{"type":"suggestion.accept","by":"ai:your-agent","suggestionId":"<id>"}` | Accept a suggestion |
| `suggestion.reject` | `{"type":"suggestion.reject","by":"ai:your-agent","suggestionId":"<id>"}` | Reject a suggestion |
| `rewrite.apply` | `{"type":"rewrite.apply","by":"ai:your-agent","content":"# Full new markdown"}` | Replace entire document |

### Create-and-apply a suggestion in one call

```json
{"type":"suggestion.add","by":"ai:your-agent","kind":"replace","quote":"old text","content":"new text","status":"accepted"}
```

### Rewrite caution

`rewrite.apply` is blocked when live collaborators are connected. Prefer `edit/v2` when anyone might have the document open. See [REFERENCE.md — Troubleshooting](./REFERENCE.md#live_clients_present-on-rewriteapply) for details.

## Choosing an Edit Strategy

| Goal | Method | Endpoint |
|---|---|---|
| Add/replace/insert blocks (recommended) | Edit V2 | `GET /snapshot` then `POST /edit/v2` |
| Simple string replacement | Structured edit | `POST /edit` |
| Replace entire document | Rewrite | `POST /ops` with `rewrite.apply` |
| Add comments or suggestions | Ops | `POST /ops` |

Start with Edit V2 for most tasks. It uses stable block refs and handles concurrent edits cleanly.

## Error Handling & Retries

| Error | Meaning | Action |
|---|---|---|
| `401/403` | Bad or missing auth | Re-read token from the URL, retry with Bearer token |
| `404` | Slug not found | Verify slug and `$BASE_URL` |
| `409 STALE_REVISION` | Snapshot out of date | Response includes latest snapshot — re-read blocks and retry |
| `409 LIVE_CLIENTS_PRESENT` | Rewrite blocked by active collaborators | Use `edit/v2` instead, or poll `/state` until `connectedClients === 0` |
| `409 ANCHOR_NOT_FOUND` | Quote text not found in document | Re-read state and use exact text from the document |
| `409 PROJECTION_STALE` | Projection metadata catching up | Re-read `state` or `snapshot`, then retry |
| `422` | Invalid payload | Fix schema and required fields |
| `422 IDEMPOTENCY_KEY_REQUIRED` | Mutation requires idempotency key | Add `Idempotency-Key` header |
| `429` | Rate limit | Back off with jitter and retry |

Always re-read state before retries that depend on anchors or revisions.

## Presence and Events

### Poll for changes

```bash
curl -sS "$BASE_URL/documents/$SLUG/events/pending?after=0&limit=100" \
  -H "Authorization: Bearer $TOKEN"
```

### Acknowledge processed events

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/events/ack" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"upToId": 42, "by": "ai:your-agent"}'
```

### Send presence

```bash
curl -sS -X POST "$BASE_URL/documents/$SLUG/presence" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Agent-Id: your-agent" \
  -d '{"agentId":"your-agent","status":"thinking","summary":"Reviewing section 2"}'
```

## Discovery

```bash
# Agent capabilities and API surface
curl -sS "$BASE_URL/.well-known/agent.json"

# Full API documentation
curl -sS "$BASE_URL/agent-docs"
```

## Advanced Reference

For route aliases, `/edit` structured operations, convergence fields, collab lifecycle, troubleshooting, and environment auth modes, see the full reference:

[REFERENCE.md](./REFERENCE.md)
