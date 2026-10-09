# Metrics batch schema v1

Planned (#209): the JSON Schema for the aggregated daily batch, field documentation and examples.
Categories: OCR, search, credits tuning and environment. Additive evolution within a major;
clients re-ask for consent when a category is added (#213).

The never-collected list in the root [README](../../README.md) is a hard constraint: no field may
carry content or identifiers, and a review must confirm it. Contract tests are shared by the
client and the collector.
