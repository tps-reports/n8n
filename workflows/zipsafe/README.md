# ZipSafe n8n Notification Workflows

These workflows replace the standalone Inspectomatic `notification_service` by wiring ZipSafe MQTT events to FiveX's existing Twilio and Email infrastructure.

## Workflows

| File | Trigger | Purpose |
|------|---------|---------|
| `manufacturer-48hr-alert.json` | MQTT `zipsafe/incident/notification` | Emails manufacturer within 48 hours of an incident |
| `overdue-inspection-alert.json` | Schedule — daily 6:00 AM | Emails facility admin + SMS inspector for every overdue inspection |
| `equipment-lifecycle-reminder.json` | Schedule — daily 7:00 AM | Emails facility admin at 180/30/7-day thresholds; SMS for 7-day only |
| `inspection-complete-notification.json` | MQTT `zipsafe/inspection/complete` | Emails facility admin with pass/fail summary |
| `compliance-violation-alert.json` | MQTT `zipsafe/compliance/alert` | SMS + email facility admin and platform admin for critical violations only |

## Prerequisites

### n8n Credentials (configure in n8n Settings → Credentials)

| Credential Name | Type | Used By |
|-----------------|------|---------|
| `ZipSafe PostgreSQL` | PostgreSQL | All DB nodes |
| `ZipSafe SMTP` | SMTP | All email nodes |
| `Twilio Basic Auth` | HTTP Basic Auth | All SMS nodes — username: `$TWILIO_ACCOUNT_SID`, password: `$TWILIO_AUTH_TOKEN` |

### n8n Environment Variables (configure in n8n Settings → Variables)

| Variable | Description |
|----------|-------------|
| `TWILIO_ACCOUNT_SID` | Twilio account SID |
| `TWILIO_FROM_NUMBER` | Twilio sending phone number (E.164 format, e.g. `+15005550006`) |
| `ZIPSAFE_PLATFORM_ADMIN_EMAIL` | Platform admin email for critical violation alerts |

### MQTT Broker

The MQTT trigger nodes connect to the FiveX Mosquitto broker. Configure the MQTT credential in n8n with:
- **Host**: `localhost` (dev) or `mqtt.coyoteforge.com` (prod)
- **Port**: `1883` (dev) or `8883` (prod, TLS)
- **Username**: JWT token from FiveX auth
- **Password**: any non-empty placeholder

### Database Schema

All SQL queries reference the `zipsafe` schema. Required tables:

```
zipsafe.equipment
zipsafe.manufacturers
zipsafe.facilities
zipsafe.facility_admins
zipsafe.users
zipsafe.inspection_schedules
zipsafe.manufacturer_notifications
```

The `manufacturer_notifications` table must exist before activating `manufacturer-48hr-alert`. If it does not yet exist, run:

```sql
CREATE TABLE IF NOT EXISTS zipsafe.manufacturer_notifications (
    id               BIGSERIAL PRIMARY KEY,
    incident_id      BIGINT NOT NULL,
    manufacturer_id  BIGINT,
    equipment_id     BIGINT NOT NULL,
    recipient_email  TEXT,
    recipient_name   TEXT,
    delivery_status  TEXT NOT NULL DEFAULT 'pending',
    error_message    TEXT,
    sent_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Importing Workflows

### Via n8n UI

1. Open n8n at `http://localhost:5678`
2. Go to **Workflows** → **Import from file**
3. Select the JSON file
4. After import, open each workflow and assign credentials to each node that requires them
5. Activate the workflow using the toggle in the top-right corner

### Via n8n REST API

```bash
N8N_BASE=http://localhost:5678
N8N_API_KEY=your-api-key

for f in /path/to/n8n/workflows/zipsafe/*.json; do
  curl -s -X POST "$N8N_BASE/api/v1/workflows" \
    -H "X-N8N-API-KEY: $N8N_API_KEY" \
    -H "Content-Type: application/json" \
    -d @"$f"
done
```

After import via API, activate each workflow:

```bash
WORKFLOW_ID=your-workflow-id
curl -X PATCH "$N8N_BASE/api/v1/workflows/$WORKFLOW_ID" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"active": true}'
```

## MQTT Payload Schemas

### `zipsafe/incident/notification`

```json
{
  "incident_id": 42,
  "equipment_id": 17,
  "facility_id": 3,
  "incident_type": "equipment_failure",
  "severity": "high",
  "description": "Harness strap failure during use.",
  "reported_at": "2026-04-08T09:00:00Z",
  "reporter_name": "Jane Smith"
}
```

### `zipsafe/inspection/complete`

```json
{
  "inspection_id": 101,
  "equipment_id": 17,
  "facility_id": 3,
  "result": "fail",
  "passed": false,
  "inspector_id": 8,
  "inspector_name": "Bob Jones",
  "completed_at": "2026-04-08T14:30:00Z",
  "deficiencies": ["Strap fraying at buckle", "Label unreadable"],
  "notes": "Remove from service until repaired.",
  "next_due_date": "2026-10-08"
}
```

### `zipsafe/compliance/alert`

```json
{
  "violation_id": 55,
  "facility_id": 3,
  "equipment_id": 17,
  "severity": "critical",
  "violation_type": "inspection_overdue_critical",
  "regulation_code": "ANSI Z359.1",
  "description": "Fall protection equipment 90 days past mandatory inspection.",
  "detected_at": "2026-04-08T06:05:00Z"
}
```

## Notes

- All email nodes require the **ZipSafe SMTP** credential to be configured with a real SMTP server
- Twilio HTTP nodes use HTTP Basic Auth: username = `TWILIO_ACCOUNT_SID`, password = `TWILIO_AUTH_TOKEN`
- The compliance workflow silently drops non-critical alerts after the severity check — this is intentional
- The lifecycle reminder queries for equipment due in exactly 7, 30, or 180 days so each item is notified only once per threshold
