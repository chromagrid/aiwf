# AIWF v11 — AI Workflow Framework (for n8n)

AIWF v11 is a **fail‑fast, production‑grade workflow** for **n8n** that turns free‑form requests into composable automations using a **rule → embeddings → LLM** selector stack, **Redis** for catalogs, **strict JSON validation**, **per‑user policy**, **idempotency**, and **observability**.

---

## TL;DR

- **Endpoint:** `POST /aiwf/v11` (plus `OPTIONS /aiwf/v11` preflight)
- **Core:** Auth → RL → Context → Catalog → (Embeddings) → Rule Select → (LLM) → Compose → Finalize
- **Storage:** Redis REST for catalogs, rate limit, idempotency, and user policy
- **Sane defaults:** CORS headers, Retry‑After, response clamping, debug timing, degraded-mode switch

---

## Features

- **Fail‑fast pipeline**: Validation → Auth → Rate‑Limit → Env/Subflow checks → Catalog; early exits return `4xx/5xx` with details.
- **OPTIONS preflight**: Instant `204` with CORS headers (no 405s, reduced browser latency).
- **Hard ENV gates**: If `REDIS_REST_URL/TOKEN` missing → skip catalog fetch & mark `catalog_unavailable`. If `OPENROUTER_API_KEY` missing → force `llm_selector_enabled=false`.
- **Idempotency**: Honors `Idempotency-Key` header. Caches `statusCode` + response body in `aiwf:idemp:<key>`, TTL‑bound.
- **Per‑user policy**: Fetch `aiwf:user:<user_id>` to apply `{ rl_max, embed_cap, llm_enabled, max_nodes, allow_dest:[…] }`. Enforce forbidden dests with `403`.
- **Selector stack**: Token/tag rule scoring (with weights) → optional embeddings top‑K boost → optional LLM strict‑JSON select if below threshold.
- **Strict LLM output**: JSON‑only, fenced‑block tolerant, schema‑validated, and `pattern_id` whitelisted against the presented `llm_catalog`.
- **Degraded mode**: Flag to bypass embeddings/LLM on infra issues. Propagates `meta.degraded=true`.
- **Observability**: `trace_id` + per‑phase timings: `validate/auth/catalog/select/llm/compose/finalize`.
- **Response clamp**: Strips bulky internals (`id2pattern`, `tag2ids`, `llm_catalog`, optional `workflow`) if payload > `RESPONSE_MAX_KB`.
- **Safe extras**: Input `extras` sanitized to a small, known schema with max length.
- **Multi‑bucket RL**: User + IP + endpoint path buckets.

---

## API

### CORS / Preflight
`OPTIONS /aiwf/v11` → `204 No Content`  
Headers include:
- `Access-Control-Allow-Origin: *` (override with `CORS_ALLOW_ORIGIN`)
- `Access-Control-Allow-Headers: Authorization, Content-Type, X-Requested-With, Idempotency-Key`
- `Access-Control-Allow-Methods: POST, OPTIONS`
- `Access-Control-Max-Age: 3600`

### Main
`POST /aiwf/v11`

**Request Headers**
- `Authorization: Bearer <API_KEY>` (if `REQUIRE_API_KEY=true`)
- `Idempotency-Key: <opaque-string>` (optional but recommended)

**Request Body (minimal)**
```json
{
  "workflow_description": "Webhook → save payload to Google Drive as JSON",
  "variant_trigger": "webhook",
  "variant_dest": "drive",
  "extras": [
    { "dest": "slack" },
    { "if": { "expr": "$json.priority === 'high'", "then": { "dest": "slack" }, "else": { "dest": "sheets" } } }
  ],
  "api_key": "optional-if-sent-in-header"
}
```

**Response (example)**
```json
{
  "status": "ready",
  "pattern_id": "webhook_to_drive",
  "used_llm": false,
  "rule_score": 7.5,
  "candidate_count": 3,
  "node_count": 5,
  "filename": "webhook-drive-2025-08-09T12-00-00.json",
  "drive_link": null,
  "tested": false,
  "meta": {
    "trace_id": "tr_9jm2lq3kmpc1y1",
    "degraded": false,
    "truncated": false,
    "timing": {
      "validate_ms": 2,
      "auth_ms": 5,
      "catalog_ms": 12,
      "select_ms": 3,
      "llm_ms": 0,
      "compose_ms": 11,
      "finalize_ms": 19
    }
  }
}
```

**Status Codes**
- `200` OK — Composed workflow (or tested / previewed).
- `400` Bad Request — Validation/env issues (details in `error_details`).
- `401` Unauthorized — Missing/invalid API key.
- `403` Forbidden — Violated per‑user policy (e.g., dest not allowed).
- `429` Rate Limited — Includes `Retry-After`, `X-RateLimit-*`.
- `503` Service Unavailable — RL system down or catalog missing.

---

## Environment

| Variable | Default | Notes |
|---|---|---|
| `CORS_ALLOW_ORIGIN` | `*` | Set to your site origin in prod. |
| `REQUIRE_API_KEY` | `true` | Enforce `Authorization: Bearer` or `api_key` field. |
| `REDIS_REST_URL` | — | Upstash/Valkey REST URL. If missing, catalog disabled. |
| `REDIS_REST_TOKEN` | — | Bearer token for Redis REST. |
| `OPENROUTER_API_KEY` | — | Enables LLM steps when present. |
| `REFERRER_URL` | `https://example.com` | Only sent to OpenRouter if not default. |
| `RL_MAX` | `60` | Default per‑bucket limit. User policy can override. |
| `RL_WINDOW_MS` | `60000` | Sliding window, ms. |
| `EMBED_MODEL` | `openai/text-embedding-3-small` | Default embedding model. |
| `EMBED_CATALOG_CAP` | `50` | Max catalog items to embed. |
| `EMBED_BUDGET_MAX` | `32` | Max vectors per call (incl. query). |
| `RESPONSE_MAX_KB` | `128` | Size clamp threshold. |
| `IDEMP_TTL_S` | `900` | Idempotency cache TTL (seconds). |
| `DEGRADED` | `false` | Force degraded mode when `true`. |
| `DEFAULT_SHEET_ID` | — | Required for Sheets dest. |
| `DEFAULT_SLACK_CHANNEL` | `#alerts` | Fallback channel. |
| `DEFAULT_DRIVE_FOLDER` | — | Optional parent folder. |
| `EVENTS_SHEET_ID` | — | Optional events log sheet. |
| `SUB_AUTH_RATE_ID` | — | n8n subflow ID: Auth+RateLimit. |
| `SUB_EMBED_ID` | — | n8n subflow ID: Semantic Select. |
| `SUB_SELECT_ID` | — | n8n subflow ID: Rule Selector. |
| `SUB_COMPOSE_ID` | — | n8n subflow ID: Composer. |
| `SUB_FINALIZE_ID` | — | n8n subflow ID: Finalize+Respond. |

---

## Rate Limiting

- Sliding window ZSET pipeline: `ZREMRANGEBYSCORE`, `ZADD`, `ZCARD`, `PEXPIRE`.
- Buckets:
  - `aiwf:rl:<user_id>`
  - `aiwf:rl:ip:<ip>`
  - `aiwf:rl:path:<path>`
- Headers on 429: `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.

---

## Idempotency

- Header: `Idempotency-Key: <opaque>` (or derived from `__q_hash` + `__uid_hash` if absent).
- Key: `aiwf:idemp:<Idempotency-Key>`
- Value: JSON `{ statusCode, body }` with TTL `IDEMP_TTL_S` seconds.

---

## Per‑User Policy

- Redis key: `aiwf:user:<user_id>`
- Example value:
```json
{
  "rl_max": 30,
  "embed_cap": 16,
  "llm_enabled": true,
  "max_nodes": 80,
  "allow_dest": ["sheets","drive"]
}
```
- If `variant_dest` not in `allow_dest` → `403 Forbidden`.

---

## LLM Selection Safety

- System prompt forces **JSON‑only**.
- Post‑parse sanitizer removes code fences and extracts inner JSON.
- Validate shape: `{ pattern_id: string, params: object, confidence: number }`.
- **Whitelist**: `pattern_id` must be in the current `llm_catalog`.
- If invalid/low‑confidence → remain in rule‑based path or return `invalid`.

---

## Degraded Mode

- Set `DEGRADED=true` or auto‑detect (extend in your RL subflow).
- Skips embeddings/LLM; returns with `meta.degraded=true`.

---

## Response Clamping

- If serialized JSON exceeds `RESPONSE_MAX_KB`, drop:
  - `id2pattern`, `tag2ids`, `llm_catalog`
  - `workflow` (when `slim=true`)
- Set `meta.truncated=true`.

---

## Quickstart

1. **Import** the `AIWF v11 • API` workflow + all subflows into n8n.
2. **Set ENVs** listed above.
3. **Seed catalog** (optional) via the provided Catalog Loader or your own job.
4. **Hit the endpoint**:
```bash
curl -X POST "https://<host>/aiwf/v11" \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"workflow_description":"Webhook → Google Sheets","variant_trigger":"webhook","variant_dest":"sheets"}'
```

---

## Troubleshooting

- `503 catalog_unavailable`: Redis env missing or keys `aiwf:catalog:id2pattern` / `aiwf:catalog:tag2ids` empty.
- `401 unauthorized`: Provide `Authorization: Bearer ...` or `api_key` field; key must match `^[A-Za-z0-9-_]{16,72}$`.
- `403 forbidden`: Check `aiwf:user:<id>` policy — destination might be blocked.
- `429 rate_limited`: Respect `Retry-After`. Consider raising user policy `rl_max`.
- `drive_failed` / `sheet_id` errors: Ensure `DEFAULT_SHEET_ID` and Drive/Sheets creds configured in n8n.

---

## Changelog

### v11
- Added preflight route, idempotency, per‑user policy, degraded mode, trace/timing, response clamp, safer extras, and multi‑bucket RL.
- Hardened LLM schema validation + whitelist.

### v10
- Introduced referer gating, numeric Config values, improved logging and headers.

---

## License

MIT
