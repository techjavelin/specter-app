# SentinelOne Integration Guide

## Endpoint and Threat Telemetry Collection with SPECTER

**Document Version:** 1.0.0
**Last Updated:** 2026-06-23

---

## Overview

SPECTER can query SentinelOne management and telemetry data for a specific endpoint
and preserve the response data alongside host-based evidence. This is useful when
an examiner needs endpoint protection context, threat findings, and recent EDR
telemetry tied to the device being acquired.

The SentinelOne integration is designed to collect:

- **Agent inventory data** to confirm operating system details, agent version,
  network identifiers, and group membership.
- **Threat records** to document detections, classifications, mitigation state,
  and the event timeline associated with the endpoint.
- **Deep Visibility telemetry** for recent investigative context, including
  process creation, network, file, and registry activity.
- **Activity log entries** showing relevant console actions related to the
  device.

From a forensic perspective, this helps correlate what was observed by the EDR
platform with artifacts collected directly from disk, memory, and live system
state. It can also document whether threats were detected, mitigated, or left in
an active state at the time of collection.

---

## Prerequisites

Before enabling the integration, confirm the following:

- **Console role**: SentinelOne **Viewer** access or higher, with the ability to
  generate or receive an API token.
- **API token**: A valid API token is required for all requests.
- **Scope**: Determine whether the token is scoped at the **site** level or the
  broader **account** level.

### Site Scope vs. Account Scope

- **Site-scoped token**: Best when the examiner only needs access to devices
  within a single site. This limits exposure and is generally preferred.
- **Account-scoped token**: Appropriate when devices may reside across multiple
  sites and a centralized response team manages the environment.

If the token is site-scoped, provide the matching `site_id` in the SPECTER
configuration so the API queries target the correct tenant partition.

---

## Generating an API Token

The exact SentinelOne console layout can vary slightly by tenant version, but the
typical workflow is:

1. Sign in to the SentinelOne management console.
2. Open **Settings**.
3. Navigate to **Users**.
4. Select the user account that will be used for SPECTER access, or create a
   dedicated service account if your organization permits it.
5. Open the **API Token** section.
6. Choose **Generate** or **Regenerate** token.
7. Select the appropriate scope:
   - **Site** scope for single-site access
   - **Account** scope for broader tenant-wide access
8. Copy the generated token immediately and store it securely.
9. Record the regional API URL for the tenant and, if applicable, the `site_id`
   needed for site-scoped queries.

Treat the token as sensitive credential material. Rotate it if it is exposed,
copied into the wrong location, or no longer needed for the investigation.

---

## Configuration Reference

Configure SentinelOne under the `api_integration` section of the SPECTER
configuration file.

| Field | Required | Description |
|-------|----------|-------------|
| `provider` | Yes | Set to `sentinelone`. |
| `api_url` | Yes | SentinelOne API base URL for the tenant region. Common patterns include `https://usea1-partners.sentinelone.net`, `https://usea1.sentinelone.net`, `https://euce1.sentinelone.net`, and `https://apne1.sentinelone.net`. Use the exact tenant API URL provided by SentinelOne. |
| `api_token` | Yes | API token generated from the SentinelOne console. |
| `site_id` | Conditional | Site identifier used when queries must be constrained to a specific site. Strongly recommended for site-scoped tokens. |
| `hostname` | Yes | Exact endpoint hostname to match in SentinelOne. Use the device name as registered with the agent. |
| `options.agent_info` | No | Enable or disable collection of agent inventory information. |
| `options.threats` | No | Enable or disable collection of threat records associated with the device. |
| `options.deep_visibility` | No | Enable or disable Deep Visibility queries for recent telemetry. |
| `options.activity_log` | No | Enable or disable retrieval of console activity related to the device. |
| `options.deep_visibility_hours` | No | Time window for Deep Visibility collection. Default investigative window is typically the last `72` hours. |

### Regional API URL Examples

SentinelOne tenants are region-specific. Examples include:

- `https://usea1.sentinelone.net`
- `https://usea1-partners.sentinelone.net`
- `https://euce1.sentinelone.net`
- `https://apne1.sentinelone.net`

Use the exact URL assigned to your tenant rather than guessing from region names
alone.

---

## Data Collected

### 1. `agent_info`

Agent inventory collection is used to confirm the endpoint identity and its
registration state in SentinelOne. Typical fields include:

- Hostname
- Operating system and OS version
- SentinelOne agent version
- Local and external IP addresses, where available
- MAC addresses
- Site and group membership
- Agent connectivity and health indicators

This data helps confirm that the endpoint being examined is the same endpoint
represented in the security platform.

### 2. `threats`

Threat collection retrieves detections linked to the device, including:

- Threat identifiers
- Threat name or storyline references
- Classification or verdict
- Detection and update timestamps
- Mitigation status
- User or policy action taken
- Timeline or event history associated with the threat

This is useful for documenting whether SentinelOne observed malware, suspicious
behavior, or policy violations before the acquisition occurred.

### 3. `deep_visibility`

Deep Visibility collection focuses on recent telemetry, typically for the last
72 hours. Depending on tenant licensing and data retention, the returned records
may include:

- Process creation events
- Network connection events
- File creation, modification, or execution events
- Registry events on Windows systems

Deep Visibility data provides investigative context that may not be visible in
the local artifact set alone, especially when triaging execution timelines.

### 4. `activity_log`

Activity log retrieval captures relevant console-side actions related to the
device, such as:

- Administrative actions taken on the endpoint
- Policy changes affecting the device
- Response actions initiated from the console
- Other relevant tenant-side audit entries tied to the host

This can help establish who interacted with the endpoint in SentinelOne and when
those actions occurred.

---

## Output Format

SPECTER writes SentinelOne responses as JSON files, typically under the
integration-specific output directory for the run. Each enabled endpoint is saved
separately to preserve source structure and simplify downstream review.

Typical file names include:

- `agent_info.json`
- `threats.json`
- `deep_visibility.json`
- `activity_log.json`

Transport-layer request and response diagnostics are stored in:

- `_transport_logs/`

These transport logs are useful for auditing API behavior, troubleshooting
failed requests, and documenting throttling or timeout conditions.

---

## Rate Limits

SentinelOne API throttling varies by tenant and service tier, but many
environments encounter throttling at approximately **100 requests per minute**.

SPECTER handles HTTP `429 Too Many Requests` responses using retry and backoff
logic so collections can continue without manual intervention. Even with backoff,
large Deep Visibility queries may take longer in busy tenants.

---

## YAML Configuration Example

```yaml
api_integration:
  provider: sentinelone
  api_url: https://usea1.sentinelone.net
  api_token: ${S1_API_TOKEN}
  site_id: "1234567890123456789"
  hostname: WS-02341
  options:
    agent_info: true
    threats: true
    deep_visibility: true
    activity_log: true
    deep_visibility_hours: 72
```

If environment variable substitution is not used in your deployment workflow,
replace `${S1_API_TOKEN}` with the actual token value in a securely managed
configuration file.

---

## Troubleshooting

### 401 Unauthorized

Cause:
- Expired, revoked, or incorrect API token

Resolution:
- Generate or retrieve a valid token
- Confirm the token was copied without truncation or extra whitespace
- Verify the request is pointed at the correct tenant `api_url`

### 403 Forbidden

Cause:
- Token lacks sufficient scope for the requested data
- Site-scoped token is being used outside its authorized site

Resolution:
- Confirm the account has at least Viewer access with API usage enabled
- Verify `site_id` is correct for site-scoped access
- Use an account-scoped token if the device may reside outside the current site

### 429 Too Many Requests

Cause:
- Tenant API throttling

Resolution:
- Allow SPECTER backoff and retry logic to complete
- Reduce the breadth of Deep Visibility collection if repeated throttling occurs
- Run large acquisitions during lower-traffic periods where operationally possible

### Hostname Not Found

Cause:
- Hostname does not exactly match the SentinelOne agent record

Resolution:
- Confirm the precise endpoint name in the SentinelOne console
- Check for domain suffix differences, asset renaming, or stale agent records
- Verify the device is actually enrolled in the targeted site or account scope

### Deep Visibility Query Timeout

Cause:
- Large telemetry window or tenant-side query latency

Resolution:
- Reduce the query window if supported by your workflow
- Re-run with a narrower scope focused on the relevant time period
- Confirm the tenant has Deep Visibility licensing and data retention available

---

## Related Documentation

- [Evidence Collection Guide](../EVIDENCE-GUIDE.md)
- [Working with SPECTER Evidence Artifacts](../WORKING-WITH-EVIDENCE.md)
- [SPECTER Technical Reference](../TECHNICAL-REFERENCE.md)
