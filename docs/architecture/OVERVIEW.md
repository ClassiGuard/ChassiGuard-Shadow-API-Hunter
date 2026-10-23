# Reference architecture
```text
Windows (separate services/fault domains)
  IIS [UI] -> Class/Shadow Management API [Go Windows Service] -> SQL Server [HA options]
                           | signed config / heartbeat / status
             mutual TLS + identity + replay protocol
                           |
Linux selected nodes (systemd per component)
  connectors -> Rust ingest -> local durable spool -> event broker -> session correlator
                                             |                   -> endpoint classifier
                                             |                   -> inventory reconcile
                                             |                   -> risk engine
                                             +-> evidence vault   -> read models
  policy-sync/cache -> local signed snapshot (168h management offline autonomy)
  isolated AI gateway -> Ollama/vLLM OR approved external provider
  authorized active verifier (optional, separate worker pool)
```

Do not assume every service deployed on same host. Agents communicate through versioned protobuf/gRPC or durable events; backpressure, circuit breakers, tenant quotas and retry budgets. Central Windows API unavailable => Linux data plane remains locally autonomous. UI unavailable => no impact on engine. Broker unavailable => local spool buffers up to configured capacity with explicit degradation alerts.

Services may be installed independently subject to validated dependency plans. See SERVICE_CATALOG and FAILURE_MATRIX. Control plane and processing plane have distinct credentials and networks.
