# Versioned event/data contracts
Use protobuf + JSON schema projections; schema registry compatibility policy. Events always include `schema_version`, `event_id` (UUID), `tenant_id`, `deployment_id`, `node_id`, `source_id`, `occurred_at_utc`, `ingested_at_utc`, `trace_id` when available, `data_classification`, `policy_version` and `provenance`.

## ApiObservation v1
`protocol`, `scheme`, `host_hash_or_domain`, `port`, `http_method`, `normalized_route`, `route_fingerprint`, `status_code`, `request_size`, `response_size`, `auth_signal` (non-secret), `client_pseudonym`, `session_pseudonym`, `graphql_operation_hash`, `content_type`, `request_field_names_redacted`, `response_field_names_redacted`, `tls_observation_mode`, `source_confidence`, `observed_at`. No bearer tokens, cookie values, passwords, payment data, payload bodies by default.

## ApiAsset v1
`asset_id`, `environment`, `service_id`, `normalized_route`, `method`, `version_tag`, `owner_id`, `inventory_status`, `spec_revision`, `first_seen`, `last_seen`, `source_count`, `confidence`, `sensitivity_tags`, `change_history_ref`.

## Finding v1
`finding_id`, `rule_id`, `category`, `severity`, `confidence`, `status` (hypothesis / under_review / validated / false_positive / accepted_risk), `asset_ids`, `evidence_refs`, `reproduction_scope_id`, `policy_version`, `model_version` (if AI used), `created_at`, `updated_at`.

## Replay and ordering
At-least-once delivery, per-tenant/source partition keys, idempotency by event_id, out-of-order event tolerance, watermark and correction events. Consumers must be resilient to duplicates; avoid storing customer PII in queue partition keys. Implement schema evolution and privacy tests for every contract change.
