# VethuQ-metrics

Open-source collector, versioned schema and docs for VethuQ's **opt-in** usage metrics. Public for
transparency; **no secrets, no raw data**.

Sharing metrics is off until a user turns it on. In return the user gets a small credit bonus, and
we get the data to improve OCR and search. Everything the client can send is defined by the schema
in this repo, so anyone can check what leaves the machine.

> Status: repository set-up (#206). Describes the planned behaviour defined in issues #207 to #216;
> the schema (#209), the collector (#207) and operations (#208) are separate stories, and their
> directories hold a README that says what will land there.

## What is collected

One **aggregated daily batch** per install (counters, buckets and histograms, never individual
events or content). Categories, as defined by the planned schema v1 (#209):

- **OCR:** per phase, file type, size bucket, language and engine version: pages, duration (median,
  p95), peak memory and CPU buckets, device (CPU or GPU), confidence statistics, line counts,
  language-judging outcomes, new text added per language pass and per rotated angle, failure reason
  codes, retries, timeouts, native-text versus OCR pages, page-size, pixel and DPI buckets.
- **Search:** engine, query length bucket, query script (Latin, Telugu, mixed), settings used,
  result-count and latency buckets, zero-result rate, engine switches after a zero-result search,
  library-size bucket.
- **Tuning:** credits charged versus measured seconds per operation, and how often the daily
  credits ran out.
- **Environment:** app version, OS family, CPU and RAM buckets, GPU yes or no, install type,
  installed add-ons, execution mode.

## What is never collected

File names, paths, file or OCR content, search query text, user names, your GitHub id, licence ids,
device hashes, and IP addresses (the collector does not store them). The install id is random, is
generated when you opt in, can be rotated on request, and is **not derived from, or linked to,
your licence or device**.

The schema must be built so that no field can carry content or an identifier (a review confirms
it, #209), and the client is required to have a test proving that query text, paths and file names
cannot reach the counters (#210).

## How sharing works

1. **Opt in.** Off by default. The first-run prompt and the `metrics.share` setting control it; the
   consent text is versioned and you are asked again if the schema adds a category (#212, #213).
2. **Preview.** The client is meant to let you see exactly what would be sent before it is sent (#216).
3. **Send.** At most once a day, in the background, to the endpoint named in the signed policy.
   Failures never block startup or local work.
4. **Opt out.** Opting out stops sending immediately and clears the unsent outbox.

## Retention and deletion

- Raw batches are kept for **90 days**; aggregates are kept (#207).
- To erase your data, send a deletion request for your install id (`DELETE`, see the
  [contract](docs/contract.md)); the client and `docs/metrics.md` (#216) explain how. Deletion
  removes the batches stored for that install id.
- The hosting provider can see IP addresses in the normal course of serving requests; the collector
  itself does not store them.

## How the endpoint is configured

The client does not hard-code the collector. The signed policy delivers `metrics.enabled`,
`metrics.endpoint` and `metrics.max_batch_bytes` (and the bonus percent), so the collector can be
moved or switched off **without a client release**. Field definitions are in
`vethuq-policy` (`docs/schema-v1.md`, section `metrics`).

## Layout

| Path | Purpose | Story |
|---|---|---|
| `docs/contract.md` | The host-independent HTTP contract | #206 |
| `docs/deployment.md` | Deployment notes: principles and the v1 host (Cloudflare Workers + D1) | #206 |
| `collector/` | Collector source | #207 |
| `schema/v1/` | The metrics batch JSON Schema, docs and examples | #209 |
| `docs/metrics.md` | User-facing explanation, preview, opt-out, deletion (planned) | #216 |

## Hosting

v1 targets a free-tier edge function with free-tier storage (Cloudflare Workers + D1). The HTTP
contract does not depend on that choice, so the collector can move to AWS later (#208).

## Never commit

This repo is public. Never commit:

- secrets, tokens or keys of any kind (they live only in the hosting provider's environment)
- `.env` files and local dev variable files (`.dev.vars`)
- raw metrics data, exports or database dumps
- analysis notebooks and queries (those stay private, #217)

`.gitignore` and the pre-commit hooks block the common cases; they are a safety net, not a
substitute for care. Enable GitHub secret scanning and push protection on the repo.

## Security

See [SECURITY.md](SECURITY.md).
