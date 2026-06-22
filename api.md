# TruthStamp API (v1)

Create permanent, independently verifiable on-chain proofs from any application,
service, or agent — with a single HTTP request. Every stamp is recorded on
MegaETH and (for public stamps) stored permanently on Arweave.

- **Base URL:** `https://stlfgfaaukrgiwwkwceb.supabase.co/functions/v1/api-v1`
- **Format:** JSON request and response bodies
- **Auth:** API key (`ts_live_...`) + project anon key

> A TruthStamp proof shows that a hash of your content existed at a given time.
> It does not by itself establish authorship or legal ownership. See the
> [disclaimer](https://truthstamp.io/disclaimer.html).

---

## Authentication

Every request requires **two headers**:

| Header | Value |
|--------|-------|
| `Authorization` | `Bearer ts_live_...` — your secret API key |
| `apikey` | Your project's public anon key (required by the gateway) |

Your `ts_live_` key is secret — treat it like a password and keep it server-side.
The `apikey` value is public (the same anon key used by the web app) and only
satisfies the platform gateway. Keys are shown once at creation and stored only
as a hash; if you lose one, revoke it and create a new one.

---

## Credits

API usage draws from a dedicated API credit balance, separate from the web app.
Credits are charged **only on success** — any failure is automatically refunded.

| Operation | Cost |
|-----------|------|
| Create text stamp | 1 credit |
| Create URL stamp | 2 credits |
| Reveal text seal | 1 credit |
| Reveal URL seal | 2 credits |
| Verify content | Free |
| Bitcoin archive | Preview — no charge yet |
| File stamp / reveal | 3 credits *(not in v1)* |

---

## Supported stamp types

| Type | Status | Notes |
|------|--------|-------|
| Public text | ✅ Supported | Content stored on Arweave, publicly viewable |
| Sealed text | ✅ Supported | Hidden until a reveal date you choose |
| Hash-only | ✅ Supported | Only the fingerprint is stored; content stays with you |
| URL | ✅ Supported | URL + page snapshot recorded on Arweave |
| Bitcoin archive | ⏳ Preview | Requests queued; processing coming soon |
| File | ❌ Coming soon | Not available in API v1 |

---

## Create a stamp

```
POST /stamps
```

### Request fields

| Field | Type | Description |
|-------|------|-------------|
| `content` | string | **Required.** The text or URL to stamp. |
| `content_type` | string | `text` or `url`. Default `text`. |
| `visibility` | string | `public`, `sealed`, or `hash_only`. Default `public`. |
| `sealed_until` | number | **Required for sealed.** Unix seconds; must be ≥10 minutes in the future. |
| `display_title` | string | Optional label (shown for public/revealed stamps). |
| `tags` | string[] | Optional, up to 5 tags. |

### Example — public text

```bash
curl -X POST "https://stlfgfaaukrgiwwkwceb.supabase.co/functions/v1/api-v1/stamps" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: a-unique-id" \
  -d '{
    "content": "My first API stamp",
    "content_type": "text",
    "visibility": "public"
  }'
```

Response `200`:

```json
{
  "success": true,
  "proof_id": 20,
  "content_type": "text",
  "visibility": "public",
  "tx_hash": "0x996a09bf...2c3e66",
  "block_number": 18798410,
  "storage_uri": "ar://7iQwLOiVvoEq...zA8n0",
  "proof_url": "https://truthstamp.io/proof.html?id=20",
  "api_credits_remaining": 99,
  "sealed_until": null
}
```

### Example — sealed text

```bash
curl -X POST ".../api-v1/stamps" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "BTC hits a new ATH this cycle",
    "content_type": "text",
    "visibility": "sealed",
    "sealed_until": 1781708587,
    "display_title": "BTC prediction"
  }'
```

### Example — hash-only

```bash
curl -X POST ".../api-v1/stamps" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "secret content only I know",
    "content_type": "text",
    "visibility": "hash_only"
  }'
```

### Example — URL

```bash
curl -X POST ".../api-v1/stamps" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "https://example.com/article",
    "content_type": "url",
    "visibility": "public",
    "display_title": "Example article"
  }'
```

---

## Retrieve a stamp

```
GET /stamps/:proof_id
```

Fetch public proof data by `proof_id`. Sealed and hash-only stamps never expose
their content — sealed content stays hidden until revealed, and hash-only
content is never returned. The hash is always present so anyone can verify.

```bash
curl ".../api-v1/stamps/20" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY"
```

Response `200` (public stamp):

```json
{
  "success": true,
  "proof_id": 20,
  "content_type": "text",
  "visibility": "public",
  "revealed": false,
  "content_hash": "0x34304742...bdbc1590",
  "content": "My first API stamp",
  "display_title": "My first API stamp",
  "tx_hash": "0x996a09bf...2c3e66",
  "block_timestamp": "2026-06-16T07:37:01+00:00",
  "storage_uri": "ar://7iQwLOi...zA8n0",
  "proof_url": "https://truthstamp.io/proof.html?id=20"
}
```

For a sealed stamp before its reveal date, or any hash-only stamp, `content`
is `null`. The hash is always present.

---

## Reveal a sealed stamp

```
POST /stamps/:proof_id/reveal
```

Reveal a sealed stamp on-chain after its `sealed_until` date has passed. This
uploads the content to Arweave and records the reveal on MegaETH. After reveal,
`GET /stamps/:proof_id` returns the unsealed content.

Only the original stamp creator can reveal. Cost mirrors the underlying type:
text = 1, URL = 2, file = 3 credits. **Credits are refunded on any failure.**

```bash
curl -X POST ".../api-v1/stamps/27/reveal" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: my-reveal-001"
```

Response `200`:

```json
{
  "success": true,
  "proof_id": 27,
  "revealed": true,
  "tx_hash": "0xa490f00c...d3e3d",
  "arweave_uri": "ar://H3els0gCLLEW...jXc",
  "proof_url": "https://truthstamp.io/proof.html?id=27",
  "credits_charged": 1,
  "api_credits_remaining": 192
}
```

### Error conditions

| HTTP | error | Meaning |
|------|-------|---------|
| 403 | `not_owner` | Only the stamp creator can reveal it |
| 400 | `not_sealed` | The stamp isn't a sealed stamp |
| 400 | `still_sealed` | Reveal date hasn't passed yet |
| 409 | `already_revealed` | This stamp was already revealed |
| 402 | `insufficient_api_credits` | Not enough API credits |
| 422 | `reveal_failed` | On-chain or Arweave failure; credits refunded |

> Reveal is synchronous and typically takes 15–30 seconds. For revealing many
> stamps, loop client-side respecting your rate limit.

---

## Bitcoin archive *(Preview)*

```
POST /stamps/:proof_id/bitcoin-archive
GET  /stamps/:proof_id/bitcoin-archive
```

Embed a TSA-1 proof manifest into the Bitcoin blockchain via OP_RETURN —
independently verifiable on Bitcoin mainnet, entirely separate from
TruthStamp's infrastructure.

> **Preview:** Requests are queued immediately but Bitcoin processing is
> currently handled manually on a best-effort basis. No credits are charged
> during the preview period. Poll the GET endpoint to track status.

### POST — queue an archive request

The stamp must be public or a revealed seal. Returns immediately with a
`job_id` for status tracking.

```bash
curl -X POST ".../api-v1/stamps/27/bitcoin-archive" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: btc-archive-001"
```

Response `202`:

```json
{
  "success": true,
  "job_id": "5c856f50-e907-4ba3-ba12-349635752fcf",
  "proof_id": 27,
  "status": "queued",
  "processing_enabled": false,
  "credits_charged": 0,
  "message": "Bitcoin archive queued. Processing is currently in preview..."
}
```

### GET — check archive status

```bash
curl ".../api-v1/stamps/27/bitcoin-archive" \
  -H "Authorization: Bearer ts_live_YOUR_KEY" \
  -H "apikey: YOUR_ANON_KEY"
```

Response `200`:

```json
{
  "success": true,
  "proof_id": 27,
  "bitcoin_txid": null,
  "bitcoin_archive_status": null,
  "bitcoin_archived_at": null,
  "job": {
    "job_id": "5c856f50-e907-4ba3-ba12-349635752fcf",
    "status": "queued",
    "txid": null,
    "created_at": "2026-06-22T08:51:11.289683+00:00",
    "confirmed_at": null,
    "last_error": null
  },
  "processing_enabled": false
}
```

### Job status values

| status | Meaning |
|--------|---------|
| `queued` | Request recorded, waiting for processing |
| `broadcast` | Bitcoin TX broadcast, waiting for confirmation |
| `confirmed` | Confirmed in a Bitcoin block — `bitcoin_txid` populated |
| `failed` | Processing failed — check `last_error` |

### Error conditions

| HTTP | error | Meaning |
|------|-------|---------|
| 403 | `not_owner` | Only the stamp creator can archive it |
| 400 | `stamp_not_confirmed` | Stamp not yet confirmed on MegaETH |
| 400 | `not_archivable` | Stamp must be public or a revealed seal |
| 409 | `already_archived` | This stamp is already archived to Bitcoin |

---

## Idempotency

Send an `Idempotency-Key` header with a unique value (e.g. a UUID) per logical
request. If a request with the same key already succeeded, the original response
is returned and no new operation is performed or charged.

```
-H "Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7"
```

Keys are remembered for 24 hours. Only successful responses are stored, so
failed requests can be retried safely. Supported on:
`POST /stamps`, `POST /stamps/:id/reveal`, `POST /stamps/:id/bitcoin-archive`.

---

## Rate limits

Each API key has a per-minute request limit (default **30/min**). Exceeding it
returns `429 rate_limited`. Need a higher limit? Get in touch.

---

## Error codes

| HTTP | error | Meaning |
|------|-------|---------|
| 401 | `missing_api_key` | No Bearer key provided |
| 401 | `invalid_api_key` | Key is wrong or revoked |
| 402 | `insufficient_api_credits` | Not enough API credits |
| 400 | `missing_content` | `content` field is empty |
| 400 | `invalid_url` | URL must start with `http://` or `https://` |
| 400 | `invalid_visibility` | `visibility` not recognised |
| 400 | `unsupported_content_type` | Only `text` and `url` in v1 |
| 400 | `missing_sealed_until` | Sealed stamp without a reveal date |
| 400 | `seal_too_soon` | Reveal date under 10 minutes away |
| 409 | `duplicate_content` | This exact content was already stamped |
| 429 | `rate_limited` | Per-minute request limit exceeded |
| 422 | `stamp_failed` | Stamping failed; credits refunded |
| 403 | `not_owner` | Reveal / archive: only the creator can do this |
| 400 | `not_sealed` | Reveal: stamp isn't sealed |
| 400 | `still_sealed` | Reveal: reveal date hasn't passed yet |
| 409 | `already_revealed` | Reveal: stamp was already revealed |
| 422 | `reveal_failed` | Reveal: failed; credits refunded |
| 400 | `not_archivable` | Archive: stamp must be public or revealed |
| 409 | `already_archived` | Archive: stamp already archived to Bitcoin |

> On any failure during stamp creation or reveal, your API credits are
> automatically refunded — you are never charged for an operation that did
> not succeed.

---

## License

MIT
