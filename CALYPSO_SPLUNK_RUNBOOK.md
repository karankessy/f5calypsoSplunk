# CalypsoAI + Splunk HEC Runbook

## 1) Environment Snapshot

- Deployment type used for validation: self-managed Splunk Enterprise
- Splunk version validated: `10.2.1`
- Splunk home (typical): `/opt/splunk`
- Splunk Web (example): `http://<splunk-host>:8000`
- Splunkd management API (example): `https://<splunk-host>:8089`
- HEC endpoint (example): `https://<splunk-host>:8088/services/collector`

## 2) HEC Configuration (Observed)

Global HEC is enabled.

Observed config stanzas/files:
- `$SPLUNK_HOME/etc/system/local/inputs.conf`
- `$SPLUNK_HOME/etc/apps/splunk_httpinput/default/inputs.conf`

Observed key values:
- `[http]`
- `disabled = 0`
- `port = 8088`
- `enableSSL = 1`

Important note:
- Token stanzas like `[http://<token_name>]` were not visible in the accessible config output. Token values may be in restricted or app-private locations.

## 3) How To Get/Create HEC Token

If token is not already known, create or retrieve it from Splunk Web:

1. Go to `Settings -> Data Inputs -> HTTP Event Collector`.
2. Create a new token (or inspect an existing one).
3. Capture:
   - Token name
   - Token value
   - Default index
   - Allowed indexes
   - Default sourcetype (if set)
4. Keep the token secret.

Optional CLI/API approach (admin creds required):
- `POST /servicesNS/nobody/splunk_httpinput/data/inputs/http`
- Requires Splunk management auth on `8089`.

## 4) TLS/Certificate Compatibility Guidance (Important)

Some Splunk Cloud endpoints can fail Calypso integration when the presented certificate chain (root CA/intermediate CA) is not trusted by the Calypso integration path.

Common symptom:
- HEC calls fail with TLS/certificate validation errors even when token and endpoint are correct.

Why this happens:
- Calypso validates the full server certificate chain.
- If Splunk Cloud serves a CA/intermediate chain not currently trusted by Calypso, TLS handshake fails.

Reliable approach:
- Use a self-managed Splunk endpoint with a broadly trusted public certificate chain.
- Certificates issued by Let’s Encrypt are commonly compatible and reduce trust-chain issues.

What to verify before integrating:
1. Endpoint serves full chain (server cert + intermediate).
2. Certificate SAN/CN matches the HEC hostname.
3. Certificate is not expired and uses modern TLS.
4. `curl`/OpenSSL validation succeeds from your integration network.

Certificate validation command examples:

```bash
openssl s_client -connect <splunk-host>:8088 -servername <splunk-host> -showcerts
```

```bash
curl -v https://<splunk-host>:8088/services/collector/health
```

If Splunk Cloud is mandatory:
- Open a support case with Splunk Cloud to confirm complete chain delivery and accepted trust anchors.
- Share the full certificate chain details with Calypso support for trust verification.

## 5) Calypso Parser/App Configuration (Observed)

A Calypso app was found:
- App: `calypsoai_app`

Observed parser settings in app `props.conf` for sourcetype `[calypsoai]`:
- `KV_MODE = json`
- `TIME_PREFIX = "time":`
- `LINE_BREAKER = ([\r\n]+)\{"source":"calypsoai"`

Observed field aliases/evals include fields like:
- `event_type`
- `event_id`
- `user_input`
- `project_id`
- `scanner_outcome`
- eval-like derived fields such as `scanner_failed_count`, `has_failures`

Interpretation:
- Events are expected to be JSON.
- Time extraction key is `time`.
- Splitting assumes event boundaries around records containing `"source":"calypsoai"`.

## 6) Dashboard Assets (Observed)

Observed dashboard view:
- `$SPLUNK_HOME/etc/apps/calypsoai_app/local/data/ui/views/calypsoai_overview.xml`

Dashboard ID/title:
- ID: `calypsoai_overview`
- Title: `CalypsoAI - GenAI Security Overview`

Observed query conventions inside dashboard:
- Uses `index=main`
- Uses source filters: `source="f5-calypso-SIQ"` OR `source="calypsoai"`

## 7) Recommended Ingestion Contract For Calypso Events

Use consistent metadata for reliable parsing and dashboard compatibility.

Recommended HEC metadata:
- `index`: `main` (or a dedicated index, then update dashboard searches)
- `sourcetype`: `calypsoai`
- `source`: `calypsoai` (or `f5-calypso-SIQ` if intentionally matching existing dashboards)

Recommended event body shape (JSON):
- Include `time` (epoch seconds or compatible timestamp)
- Include stable keys for event and scanner outcomes

Example payload:

```json
{
  "time": 1776501123,
  "source": "calypsoai",
  "event": {
    "event_id": "evt-12345",
    "event_type": "prompt_scan",
    "project_id": "proj-001",
    "user_input": "Summarize this report",
    "scanner_outcome": "pass"
  }
}
```

## 8) HEC Test Commands

Basic connectivity test:

```bash
curl -v https://<splunk-host>:8088/services/collector/health
```

Send test event:

```bash
curl -v https://<splunk-host>:8088/services/collector \
  -H "Authorization: Splunk <HEC_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"time":1776501123,"source":"calypsoai","sourcetype":"calypsoai","index":"main","event":{"event_id":"evt-12345","event_type":"prompt_scan","project_id":"proj-001","user_input":"Summarize this report","scanner_outcome":"pass"}}'
```

Search validation in Splunk:

```spl
index=main source="calypsoai" sourcetype="calypsoai"
| stats count by event_type scanner_outcome
```

## 9) Role/Permission Guidance

Observed role config appears close to standard Splunk roles (`admin`, `power`, `user`) without obvious custom Calypso roles.

Minimum practical permissions:
- Dashboard user:
  - App read access to `calypsoai_app`
  - Search capability on target index (`main` or your custom index)
- Parser/admin operator:
  - Ability to manage data inputs (HEC)
  - Ability to edit app configs (`props.conf`, `transforms.conf`, saved searches, views)
  - Restart Splunk when required by config changes

## 10) If You Move To A Dedicated Index

If you switch from `main` to something like `calypso`:

1. Update HEC token default/allowed indexes.
2. Update dashboard SPL from `index=main` to `index=calypso`.
3. Validate app permissions include read/search on new index.
4. Backfill or dual-write during migration window if dashboards must remain continuous.

## 11) Operational Checklist

- HEC port 8088 reachable from Calypso platform network.
- Valid token assigned to intended index.
- Event metadata aligned (`index`, `source`, `sourcetype`).
- Sourcetype parser for `calypsoai` active.
- Dashboard searches aligned to actual index/source values.
- Role permissions validated for viewer and admin personas.
- End-to-end test performed (send event -> search -> dashboard panel update).
- TLS chain validated against Calypso-supported trust anchors.

## 12) Gaps/Follow-ups

Items not fully extracted from read-only host audit:
- Exact active HEC token stanza names and token values.
- Full saved search inventory tied to Calypso app.

To close gaps quickly:
- In Splunk Web, export HEC input list and token settings.
- In app context, export dashboard JSON/XML and saved searches.
- Capture a known-good sample event set from Calypso production traffic for parser regression tests.
- Keep this runbook free of hostnames, IPs, usernames, and token data before sharing publicly.
