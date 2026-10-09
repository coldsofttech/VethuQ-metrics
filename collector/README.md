# Collector

Planned (#207): the `POST /v1/metrics` and `DELETE /v1/metrics/{install_id}` endpoint, with schema
validation, size and rate limits, outlier quarantine, storage and the retention job, following
[docs/contract.md](../docs/contract.md). Tests cover valid, invalid, oversize, replay,
rate-limited and delete cases.

No secrets and no real data in this directory; configuration comes from the environment (see
[docs/deployment.md](../docs/deployment.md)).
