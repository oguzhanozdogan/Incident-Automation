# SecOps Enrichment App (n8n)

Automated IOC enrichment pipeline built on n8n. It ingests security alerts, extracts indicators (IP/domain/URL), enriches them with VirusTotal and urlscan.io, aggregates results, and pushes enrichment context to an incident/ticket endpoint.

## What this app does

- Extracts IOCs from alert payload text (`ip`, `domain`, `url`)
- Runs VirusTotal and urlscan lookups in parallel branches
- Normalizes and scores results into low/medium/high/unknown verdicts
- Creates a standardized enrichment payload + markdown comment
- Sends enrichment output to your ticketing webhook (Jira/ServiceNow/GitHub adapter)
- Responds to webhook caller with execution summary

## Project structure

- `docker-compose.yml`: local n8n runtime
- `.env.example`: environment variable template (no secrets)
- `workflows/secops-enrichment-workflow.json`: importable workflow
- `examples/sample-alert.json`: test webhook payload

## Prerequisites

- Docker + Docker Compose
- Node.js 20+ (only if running n8n without Docker)
- VirusTotal API key
- urlscan API key
- A ticketing webhook/API endpoint (custom adapter or direct integration endpoint)

## Quick start

1. Create env file:

```powershell
Copy-Item .env.example .env
```

2. Fill required values in `.env`:
- `VIRUSTOTAL_API_KEY`
- `URLSCAN_API_KEY`
- `TICKETING_WEBHOOK_URL`
- optional: `TICKETING_API_TOKEN`

3. Start n8n:

```powershell
docker compose up -d
```

4. Open n8n at `http://localhost:5678`

5. Import workflow:
- Import `workflows/secops-enrichment-workflow.json`
- Activate workflow after verifying env values

## Webhook endpoint

- Path: `/webhook/secops/enrich` (active workflow)
- Method: `POST`

Local test call:

```powershell
Invoke-RestMethod -Uri "http://localhost:5678/webhook/secops/enrich" -Method Post -ContentType "application/json" -InFile "examples/sample-alert.json"
```

## Expected incoming payload

Any JSON is accepted; IOC extraction scans common text fields (`alert`, `message`, `description`, `details`, `raw`, `title`) and falls back to full payload text.

Recommended fields:

```json
{
  "incidentId": "INC-2026-00421",
  "source": "siem",
  "title": "Suspicious outbound callback",
  "description": "Host contacted hxxp://malicious-example.test/dropper and 185.220.101.1"
}
```

## Ticketing integration notes

`Update Incident Ticket` is intentionally generic and posts JSON to `TICKETING_WEBHOOK_URL`.

Use one of these patterns:
- direct API endpoint in Jira/ServiceNow/GitHub if available
- lightweight adapter service that transforms the payload into your ticketing API schema

## Security controls

- API keys are loaded from environment variables
- No sensitive credentials are hardcoded in workflow JSON
- HTTPS is used for all external intelligence requests
- Keep `.env` private and rotate API keys regularly

## Reliability and rate limiting

- `continueOnFail` enabled for external API requests
- automatic retries configured on outbound HTTP nodes
- configurable throttling via:
  - `VT_DELAY_MS`
  - `URLSCAN_DELAY_MS`

## Customization points

- IOC regex + extraction logic: `Extract IOCs` code node
- scoring model and verdict thresholds: `Normalize Enrichment` code node
- outbound ticket format: `Build Ticket Payload` code node
- target ticket API call details: `Update Incident Ticket` HTTP node

## Known limitations

- Domain extraction is regex-based and may include benign infrastructure domains
- URL reputation coverage depends on provider data availability
- For strict enterprise rate limits, extend throttling or add queueing
