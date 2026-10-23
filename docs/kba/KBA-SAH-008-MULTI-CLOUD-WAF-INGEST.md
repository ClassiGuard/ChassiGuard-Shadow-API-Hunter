# KBA-SAH-008: Multi-Cloud WAF & API Gateway Log Ingestion Integration

**Article ID:** KBA-SAH-008  
**Category:** Telemetry Ingestion & Cloud Gateways  
**Severity:** Tier 2 (Medium)  
**Target Roles:** Cloud Platform Engineers, DevOps, SecOps  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+ (`shadowapi-collector` Multi-Cloud Gateway Adapter)  

---

## 1. Overview & Architecture

Modern enterprises route public and partner traffic through managed cloud API gateways and Web Application Firewalls (WAF). Rather than requiring inline agent sidecars in every microservice container, Shadow API Hunter ingests native cloud access logs through dedicated endpoints on the collector:

```text
[ AWS API Gateway / WAF ]    ΓöÇΓöÇ> CloudWatch Logs / Firehose ΓöÇΓöÇ> POST /api/v1/collector/gateway/aws
[ Azure API Management ]     ΓöÇΓöÇ> Azure Monitor / Event Hub  ΓöÇΓöÇ> POST /api/v1/collector/gateway/azure
[ Cloudflare Logpush & WAF ] ΓöÇΓöÇ> HTTP Destination (NDJSON)   ΓöÇΓöÇ> POST /api/v1/collector/gateway/cloudflare
[ Multi-Cloud Batch Stream ] ΓöÇΓöÇ> Event Bridge / Fluent Bit  ΓöÇΓöÇ> POST /api/v1/collector/gateway/batch
```

---

## 2. Source Authentication Credentials

All gateway ingestion requests must include registered source credentials in headers:
- `X-ShadowAPI-Source-ID`: Source identifier (e.g. `src-cloud-gateway-01`, `src-aws-apigw-01`, `src-azure-apim-01`, `src-cloudflare-logpush-01`)
- `X-ShadowAPI-Source-Key`: Pre-shared secret token configured during node provisioning (e.g. `sec-cloud-gateway-2026`)

---

## 3. Configuration & Forwarding Setup

### Option 1: AWS API Gateway / CloudWatch Logs

Configure CloudWatch Log Group Subscription Filter or Kinesis Data Firehose HTTP destination:

1. Target Endpoint: `https://collector.corp.internal:9101/api/v1/collector/gateway/aws`
2. Access Log Format (JSON):
```json
{
  "requestId": "$context.requestId",
  "extendedRequestId": "$context.extendedRequestId",
  "ip": "$context.identity.sourceIp",
  "caller": "$context.identity.caller",
  "user": "$context.identity.user",
  "requestTime": "$context.requestTime",
  "requestTimeEpoch": "$context.requestTimeEpoch",
  "httpMethod": "$context.httpMethod",
  "resourcePath": "$context.resourcePath",
  "path": "$context.path",
  "status": "$context.status",
  "protocol": "$context.protocol",
  "responseLength": "$context.responseLength",
  "apiId": "$context.apiId",
  "domainName": "$context.domainName",
  "wafAction": "$context.waf.action"
}
```

---

### Option 2: Azure API Management (APIM)

Configure Diagnostic Settings in Azure Portal or via Azure Resource Manager (ARM):

1. Target Endpoint: `https://collector.corp.internal:9101/api/v1/collector/gateway/azure`
2. Enable Category: `GatewayLogs`
3. Event Payload (Standard Azure Monitor Wrapper):
```json
{
  "records": [
    {
      "operationName": "Microsoft.ApiManagement/GatewayLogs",
      "time": "2026-10-22T10:00:00.000Z",
      "correlationId": "azure-corr-id-99",
      "method": "POST",
      "url": "https://api.contoso.azure-api.net/v1/customers",
      "responseCode": 201,
      "responseSize": 1024,
      "callerIpAddress": "203.0.113.55",
      "serviceId": "contoso-apim",
      "apiId": "customers-api"
    }
  ]
}
```

---

### Option 3: Cloudflare Logpush (HTTP Requests & WAF)

Configure Cloudflare Logpush via Dashboard or API targeting the collector:

1. Target URL: `https://collector.corp.internal:9101/api/v1/collector/gateway/cloudflare`
2. Format: Newline-Delimited JSON (`application/x-ndjson`)
3. Selected Fields:
   - `RayID`, `ClientIP`, `ClientRequestHost`, `ClientRequestMethod`, `ClientRequestURI`, `EdgeResponseStatus`, `EdgeResponseBytes`, `EdgeStartTimestamp`, `WAFAction`
4. Custom Headers:
   - `X-ShadowAPI-Source-ID: src-cloudflare-logpush-01`
   - `X-ShadowAPI-Source-Key: sec-cloudflare-logpush-2026`

---

## 4. Verification & Health Monitoring

Verify gateway ingest telemetry on the collector:

```bash
# Check Prometheus Metrics for Gateway Adapters
curl -s http://127.0.0.1:9101/metrics | grep gateway_logs

# Expected Output:
# "gateway_logs_received_total": 45210
# "gateway_logs_ingested_total": 45210
# "gateway_logs_dropped_total": 0
```

If `gateway_logs_dropped_total` increments:
- Inspect `/tmp/gw_collector.log` for JSON parsing errors or malformed payloads.
- Ensure upstream forwarder sends valid JSON or NDJSON format.
