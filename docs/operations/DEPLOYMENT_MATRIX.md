# Deployment patterns
**Pilot:** one Windows management node + Windows SQL Server + one Linux edge node with selected services + local Ollama; explicitly non-HA.
**Enterprise:** Windows UI/API services and SQL HA, 3+ Linux ingest/session nodes with broker and ClickHouse cluster sized to workload, optional isolated AI GPU nodes, separate active verification workers.
**Air-gapped:** all inference/data/local PKI entirely on-prem, artifact mirror, signed offline update path, vendor cloud egress disabled.
**DMZ-separated:** passive sensors/collectors close to apps, no inbound mgmt->sensor unless explicitly approved, mutual TLS outbound sync via controlled firewall rules.

Each service has its own placement, ports, resource requests, node trust, upgrade strategy and local minimums. Support bare-metal/VM systemd Linux tarball first; Kubernetes Helm optional later. Use deployment validation to confirm no 'all-in-one' mandatory daemon.
