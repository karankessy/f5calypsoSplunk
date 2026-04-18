# CalypsoAI Splunk App (Reusable)

![Calypso Splunk Overview](f5CalypsoSplunk.png)

This app contains reusable parser and dashboard configuration for CalypsoAI logs ingested through Splunk HEC.

## Privacy/Sanitization Rule

Before publishing this repository, do not include:

- IP addresses
- Hostnames
- Usernames
- HEC token names/values
- Internal URLs or tenant identifiers

## Included

- `default/props.conf`: sourcetype parsing and JSON extraction for `calypsoai`.
- `default/transforms.conf`: helper transform for scanner array capture.
- `default/data/ui/views/calypsoai_overview.xml`: overview dashboard.
- `default/savedsearches.conf`: starter saved searches.
- `metadata/default.meta`: object permissions.

## Expected Event Metadata

- `index`: `main` (or update dashboard token default and searches)
- `sourcetype`: `calypsoai`
- `source`: `calypsoai` (or `f5-calypso-SIQ`)

## Sample HEC Event

```json
{
  "time": 1776501123,
  "source": "calypsoai",
  "sourcetype": "calypsoai",
  "index": "main",
  "event": {
    "event_id": "evt-12345",
    "event_type": "prompt_scan",
    "project_id": "proj-001",
    "user_input": "Summarize this report",
    "scanner_outcome": "pass"
  }
}
```

## Install

1. Copy `calypsoai_splunk_app` into `$SPLUNK_HOME/etc/apps/`.
2. Restart Splunk.
3. Open app from Splunk Web and load dashboard `calypsoai_overview`.

## Splunk Cloud Certificate Compatibility

If Calypso integration fails against Splunk Cloud HEC with certificate validation errors, the issue is often trust-chain compatibility:

- The endpoint certificate chain may include a CA/intermediate not currently trusted in the Calypso integration path.
- Integration can fail even when endpoint URL and token are correct.

Recommended mitigation:

1. Validate certificate chain completeness and hostname match.
2. Prefer a self-managed Splunk HEC endpoint with a broadly trusted public CA chain.
3. Let’s Encrypt-backed certificates are often a reliable option for interoperability.

Quick checks:

```bash
openssl s_client -connect <splunk-host>:8088 -servername <splunk-host> -showcerts
```

```bash
curl -v https://<splunk-host>:8088/services/collector/health
```

If Splunk Cloud is required, coordinate chain/trust validation with both Splunk Cloud support and Calypso support.

## GitHub Push Workflow

```bash
git add calypsoai_splunk_app CALYPSO_SPLUNK_RUNBOOK.md
git commit -m "Add reusable CalypsoAI Splunk parser and dashboard app"
git push
```

## Notes

- HEC tokens are not included in this repo by design.
- If using a non-`main` index, update dashboard searches or use the index input token in the dashboard.
- Keep examples generic when contributing changes so the repo remains reusable across environments.
