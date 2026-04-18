# Marketing n8n Workflows

This directory holds the n8n workflow definitions that the marketing
module's content generation spine dispatches to. These are the
"publishing layer" — the content-generator service in `mod_node`
produces the copy, and these workflows push it out to the real
channels.

## Deployed workflows

| File | Trigger | Purpose |
|---|---|---|
| `receipy-publisher.workflow.json` | MQTT `marketing/tasks/publish/receipy` | Publishes a TikTok receipy script to TikTok, Instagram Reels, X, and the blog (per `channels` array in the task payload) |
| `newsletter-generator.workflow.json` | MQTT `marketing/tasks/publish/newsletter` | Drafts + sends a newsletter via Beehiiv |
| `meeting-prep.workflow.json` | MQTT `marketing/tasks/prep/meeting` | Builds a prospect briefing packet ahead of a discovery call |
| `content-repurposer.workflow.json` | MQTT `marketing/tasks/repurpose/content` | Takes a single piece of published content and generates variants for other channels (blog → LinkedIn article → TikTok receipy) |
| `lead-capture.workflow.json` | MQTT `marketing/tasks/capture/lead` or webhook | Captures a new lead from an inbound source, enriches via the contact-enrichment pattern, and pushes to the pipeline |
| `email-sequence.workflow.json` | MQTT `marketing/tasks/sequence/email` | Drip email follow-up sequences (post-call, post-workshop, evergreen) |
| `competitive-scan.workflow.json` (skeleton) | Weekly schedule | Scrape tracked competitor URLs, diff against last week's snapshot, publish material changes to `marketing/tasks/competitive-scan/report`. **Skeleton workflow** — see inline `skeleton: true` flag in the JSON; needs the real scrape targets + diff logic. |
| `report-generator.workflow.json` (skeleton) | Weekly schedule | Aggregate published campaigns + new pipeline leads into a weekly performance report, publish to `marketing/tasks/report/weekly`. **Skeleton workflow** — needs the real aggregation + formatting logic. |

## Source

The first 6 workflows (`receipy-publisher`, `newsletter-generator`,
`meeting-prep`, `content-repurposer`, `lead-capture`,
`email-sequence`) were copied from
`mod_node/examples/workflows/marketing/` as part of the P2-7 /
audit improvement plan step 3. The last 2
(`competitive-scan`, `report-generator`) had no templates and were
**authored as skeleton workflows** — valid n8n JSON with trigger +
pipeline structure but placeholder business logic. They are marked
with `meta.skeleton: true` and can be fleshed out via the n8n UI.

### Skeleton workflows

Skeleton workflows are runnable by n8n but their core logic is a
TODO placeholder. They exist so that:

1. The marketing module's full 8-workflow contract is present on
   disk from day one, rather than implicit or missing.
2. A developer opening the n8n UI sees every workflow name the
   design spec promised, making the scope obvious.
3. The MQTT topic contract (`marketing/tasks/.../...`) is
   documented in the skeleton's node config — implementors don't
   have to guess which topic to subscribe to.

See each skeleton's `meta.purpose` field for its intended behavior.

## Importing into a running n8n instance

n8n workflows are JSON files that can be imported via the n8n API
or the CLI. From the repo root:

```bash
# Single workflow
curl -X POST http://localhost:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d @n8n/workflows/marketing/receipy-publisher.workflow.json

# Or via the n8n CLI inside the container
docker exec -it n8n n8n import:workflow \
  --input=/workflows/marketing/receipy-publisher.workflow.json
```

## Credentials

Each workflow expects its channel-specific credentials to already
exist in n8n's credential store:

- **TikTok** — `tiktok-content-publisher` (OAuth 2.0)
- **Instagram** — `instagram-business-api` (Facebook Graph API)
- **X / Twitter** — `twitter-v2-oauth` (OAuth 2.0)
- **Beehiiv** — `beehiiv-api-key` (API key)
- **Blog** — `fivex-blog-webhook` (webhook secret)

Credentials are configured per-deployment in the n8n UI at
http://localhost:5678/credentials.

## Testing workflows locally

Use the MQTT CLI to fire a test dispatch from the command line:

```bash
mosquitto_pub -h localhost -t 'marketing/tasks/publish/receipy' -m '{
  "task_id": "test-1",
  "brand_id": "brand-acme",
  "content": {
    "subject": "Hook line",
    "body": "HOOK: ...\n\nSTORY: ...\n\nCTA: ..."
  },
  "channels": ["tiktok"]
}'
```

The workflow should appear in the n8n execution log at
http://localhost:5678/workflows/executions.

## See also

- `docs/superpowers/specs/2026-03-24-marketing-operations-module-design.md` §5 — workflow design
- `docs/audit/marketing/audit_report.md` — audit context
- `mod_node/examples/workflows/marketing/` — source templates (including 6 additional workflows not yet deployed)
