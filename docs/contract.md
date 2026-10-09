# HTTP contract (v1, draft)

The contract between the VethuQ client and the collector. It does not depend on the host: any
implementation that honours it can serve it, so the collector can move (for example from
Cloudflare Workers to AWS) without a client release. The endpoint URL is delivered by the signed
policy, never hard-coded.

Status: **draft** for coldsofttech/VethuQ-support#207 (endpoint) and #209 (batch schema). Items
marked *proposed* are not decided there and are the first things to confirm.

## Conventions

- HTTPS only. JSON bodies in UTF-8, `Content-Type: application/json`.
- The URL path carries the major version (`/v1/...`). Additive changes stay within a major; clients
  ignore unknown response fields, and the collector rejects unknown **schema** versions.
- No cookies, no client credentials, no user agent logging beyond what the host does by default.
- The collector **does not store IP addresses**. Per-IP rate limiting uses short-lived counters
  that expire quickly and never persist the address (see [deployment.md](deployment.md)).

## `POST /v1/metrics`

Submit one aggregated daily batch for one install.

Request body (envelope; the `batch` object is defined by the metrics schema, #209):

```json
{
  "schema_version": 1,
  "install_id": "<random id>",
  "batch_date": "2026-10-08",
  "batch": { }
}
```

| Field | Rule |
|---|---|
| `schema_version` | Integer. Unknown versions are rejected. |
| `install_id` | The random, rotatable install id created at opt-in. Never derived from a licence or device. At least 128 bits of randomness, encoded as a string (*proposed*: 32 lowercase hex characters). |
| `batch_date` | UTC date the batch covers. One accepted batch per install per date; a repeat is a replay (below). |
| `batch` | Validated against `schema/v1`. Counters, buckets and histograms only. |

Limits: the maximum body size comes from the signed policy (`metrics.max_batch_bytes`); the server
also enforces a **hard cap** that policy cannot raise. Oversize bodies are rejected before parsing.

### Responses

| Status | Body `status` | Meaning | Client action |
|---|---|---|---|
| `200` | `accepted` | Stored | Drop the batch from the outbox |
| `200` | `duplicate` | Same install and date already accepted (replay) | Drop the batch |
| `200` | `quarantined` | Valid shape but implausible values; held for review, not used | Drop the batch (metrics are advisory) |
| `400` | `rejected` | Failed schema validation | Drop the batch; do not retry the same body |
| `413` | `rejected` | Over the size cap | Drop the batch |
| `422` | `rejected` | Unknown `schema_version` | Drop the batch; the client should update |
| `429` | `rate_limited` | Per-install or per-IP limit | Retry later, honour `Retry-After`, with backoff |
| `5xx` | n/a | Collector or host problem | Retry with backoff; keep the batch for a limited number of days |

Response body: `{ "status": "...", "reason": "<short code, optional>" }`. `reason` is a stable
machine code (for example `schema`, `size`, `version`, `rate`), never an echo of submitted values.

*Proposed:* status codes and body shape above. #207 asks for a response that "reports
accepted/rejected so the client can retry with backoff" without fixing the format.

## `DELETE /v1/metrics/{install_id}`

Erase everything stored for an install id (batches and any derived per-install rows). Aggregates
that no longer identify an install are not affected.

| Status | Meaning |
|---|---|
| `204` | Deleted (also returned when nothing was stored, so the call does not reveal whether an id exists) |
| `429` | Rate limited; retry later |

Possession of the random install id is the only credential: it is unguessable (at least 128 bits)
and known only to the install. A rotated id is a different id; to erase older data, delete the
earlier id **before** rotating. The deletion process for requests is described in the fulfilment
runbook (#204, #216).

## Kill switch and endpoint move

Both are policy-driven, not part of this contract: `metrics.enabled = false` stops the client from
sending; `metrics.endpoint` points it elsewhere. A collector may also answer `503` to shed load;
clients treat it as a retryable failure and never block the user (#211).

## Abuse handling

The collector may block an install id, tighten limits, or drop a batch range (#208). A blocked id
receives `429` with a long `Retry-After`; it is not told it was blocked.

## Contract tests

The client and the collector share fixtures for the cases above: valid, invalid, oversize, replay,
rate-limited, unknown version and delete (#207, #209). They land with the collector and schema.
