# Service-specific troubleshooting trees
**UI not loading:** browser DNS/TLS -> IIS site state -> app bundle/static files -> reverse proxy config -> `ShadowAPI.Management.Api` health -> SQL connectivity only if API endpoints fail. Do not restart Linux core because a UI page is blank.

**New API discoveries missing:** source ingestion freshness -> collector config/credentials -> spool disk/lag -> normalizer rejection -> broker topic partition -> inventory materializer -> central sync age -> UI read model. Check each component's metrics, not a monolithic restart.

**Node appears offline in UI:** Windows Sync Service -> mTLS/cert -> gateway/network -> Linux `shadowapi-sync` -> policy generation/heartbeat; verify Linux ingest/session continue locally.

**Sensitive exposure false positives:** data-classification rule -> redaction metadata -> sample provenance -> labeling feedback -> scorer version. Never reveal full payload just to debug.

**Offline replay stuck:** local spool integrity -> sequence/checksum -> broker/management capacity -> idempotency key conflict -> sync backoff; protect evidence integrity and never blindly purge pending buffers.
