---
name: log-triage-output-conventions
description: Use when transforming application logs into structured JSON error reports.
---
1. Set the top-level JSON key `schema_version` to `2` and `generated_by` to `log-triage`.
2. Normalize every service name to lower-case and replace `-` with `_` (for example, `payment-service` becomes `payment_service`).
3. Sort the `errors` array by service, then by `timestamp_utc`, both ascending.
4. Self-check:
   - Are the required schema keys and exact values present?
   - Are service names normalized consistently?
   - Is `errors` sorted by service and then ascending `timestamp_utc`?
