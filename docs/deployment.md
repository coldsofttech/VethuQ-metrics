# Deployment notes

How the collector is deployed. **Secrets are never in this repository.** They exist only in the
hosting provider's environment (and in local, uncommitted dev files that `.gitignore` blocks).

## Principles (any host)

1. **Configuration through the environment.** Anything secret or per-deployment (database ids,
   salts, operator credentials) is an environment variable or a provider secret, read at runtime.
2. **No IP addresses stored.** If per-IP rate limiting needs state, store a keyed hash of the
   address in a counter that expires within minutes, using a key that is itself a secret in the
   environment. Never write the raw address to storage or logs.
3. **Least data.** The collector stores validated, aggregated batches keyed by install id and
   date. Reject or quarantine anything else.
4. **Retention job.** Raw batches older than 90 days are deleted on a schedule; aggregates stay.
5. **Same contract everywhere.** [contract.md](contract.md) is the interface. Moving hosts must not
   change it; the new URL is announced through the signed policy (`metrics.endpoint`).
6. **Fail small.** If the free tier is exhausted the collector answers with a retryable status; the
   client retries later and nothing blocks users (#208).

## v1 host: Cloudflare Workers + D1 (free tier)

The collector is a Worker; storage is a D1 database. Both fit the free tier at launch volumes, and
the contract keeps the choice reversible.

| Concern | Where it lives |
|---|---|
| Collector code | `collector/` (this repo), deployed with Wrangler |
| D1 database | created in the Cloudflare account; bound to the Worker as a binding |
| Secrets | `wrangler secret put <NAME>`; never in `wrangler.toml` |
| Local development | `.dev.vars` (git-ignored) |
| Scheduled retention | a Cron Trigger running the retention job |

### First-time set-up (outline; the commands are finalised with the collector, #207)

1. Create the D1 database in the Cloudflare account.
2. Add the binding to the Worker configuration. The database name and ids are deployment-specific;
   keep real ids out of the repo and supply them at deploy time.
3. Apply the migrations from `collector/` to the database.
4. Set secrets with `wrangler secret put`, for example the key used to hash IPs for rate limiting
   and any operator credential for the abuse and export endpoints.
5. Deploy, then point `metrics.endpoint` in the signed policy at the Worker URL (policy
   authoring: `vethuq-entitlements` `docs/policy-authoring.md`).
6. Test the disable path end to end with the test policy (`metrics.enabled = false`, #208).

### Free-tier limits

Watch the Worker request count, D1 writes and storage. Alerts should fire well before any limit
(#208). At the limit the collector returns a retryable status; clients back off.

## Moving to AWS later

Implement the same [contract](contract.md) (for example API Gateway + a function + a database),
migrate the aggregated data, publish the new URL through the policy, and keep the old endpoint
answering until clients have switched. The written plan is part of #208.
