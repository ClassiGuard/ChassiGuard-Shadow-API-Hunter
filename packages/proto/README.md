# ClassiGuard Shadow API Hunter — Protocol Buffer Specifications

This package defines the core Protocol Buffer v3 wire formats for passive API telemetry capture, inventory asset records, cryptographic policy snapshots, and risk evidence.

## Specifications

- `api_observation.proto`: Standardized wire schema for observed HTTP/REST, gRPC, and GraphQL requests.
- `api_asset.proto`: Inventory asset records with taxonomy classification and operational lifecycle metadata.
- `finding.proto`: Risk finding records including vulnerability categories, severity ratings, and proof manifests.
- `policy_snapshot.proto`: Cryptographically signed Ed25519 policy snapshots for 168-hour offline autonomy.

---
*ClassiGuard Shadow API Hunter — Enterprise API Security*
