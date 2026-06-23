# Microsoft Intune Integration Guide

## Device Management and Graph API Collection with SPECTER

**Document Version:** 1.0.0
**Last Updated:** 2026-06-23

---

## Overview

SPECTER can collect device-management data from Microsoft Intune by querying the
Microsoft Graph API for a specific managed device. This supplements host-based
evidence with management, compliance, software inventory, and audit context from
the organization’s MDM platform.

The Intune integration is intended to collect:

- Managed device details and compliance status
- Installed application inventory
- Applied configuration profile data
- BitLocker recovery keys for the device, when specifically authorized and
  enabled
- Audit activity related to administrative actions on the device

This information can help an examiner understand how the endpoint was managed,
whether it was compliant at the time of collection, what software was reported
by Intune, and what remote administrative actions may have occurred.

---

## Prerequisites

Before using the integration, create an Azure AD application registration and
grant the required Microsoft Graph API permissions.

### Required Permissions

| Permission | Purpose |
|------------|---------|
| `DeviceManagementManagedDevices.Read.All` | Read managed device information and compliance state |
| `DeviceManagementApps.Read.All` | Read installed and detected application inventory |
| `DeviceManagementConfiguration.Read.All` | Read device configuration profiles and related settings |
| `BitlockerKey.Read.All` | Read BitLocker recovery keys for the device; optional and requires admin consent |
| `DeviceManagementAuditLogs.Read.All` | Read Intune and device-management audit events |

Additional requirements:

- A valid **tenant ID**
- An **application (client) ID**
- A **client secret**
- **Admin consent** granted for the above permissions

---

## App Registration Walkthrough

Use the Azure Portal to create the application registration:

1. Sign in to the **Azure Portal** with an account authorized to manage app
   registrations.
2. Open **Microsoft Entra ID**.
3. Go to **App registrations**.
4. Select **New registration**.
5. Enter a name such as `SPECTER-Forensics`.
6. Choose the appropriate supported account type for your tenant.
7. Create the application.
8. Open the new app registration and record:
   - **Application (client) ID**
   - **Directory (tenant) ID**
9. Go to **Certificates & secrets**.
10. Create a **New client secret** and copy the value immediately.
11. Go to **API permissions**.
12. Add the required Microsoft Graph **Application permissions** listed above.
13. Select **Grant admin consent** for the tenant.
14. Confirm the permission status shows admin consent has been granted.

If recovery key access is not needed, omit `BitlockerKey.Read.All` to reduce
privilege.

---

## Authentication

The current integration uses the **OAuth 2.0 client credentials flow** with:

- `client_id`
- `client_secret`
- `tenant_id`

This is appropriate for unattended collection workflows where an examiner or
automation system needs to query Intune without interactive browser prompts.

SPECTER does not require delegated user sign-in for the current workflow. A
future enhancement may support **device code flow** for interactive environments,
but that is not the primary model for headless acquisitions today.

---

## Configuration Reference

Configure Intune under the `api_integration` section of the SPECTER
configuration.

| Field | Required | Description |
|-------|----------|-------------|
| `provider` | Yes | Set to `intune`. |
| `tenant_id` | Yes | Microsoft Entra tenant identifier used to request Graph API tokens. |
| `client_id` | Yes | Application (client) ID of the Azure AD app registration. |
| `client_secret` | Yes | Client secret created for the app registration. |
| `device_name` | Yes | Managed device name to match in Intune. Matching is typically based on the exact managed device name returned by Graph. |
| `options.compliance` | No | Enable or disable compliance and policy evaluation collection. |
| `options.installed_apps` | No | Enable or disable installed application inventory collection. |
| `options.config_profiles` | No | Enable or disable configuration profile collection. |
| `options.recovery_keys` | No | Enable or disable BitLocker recovery key retrieval. Use only when explicitly authorized. |
| `options.audit_logs` | No | Enable or disable audit log collection for device-related actions. |

### Device Name Matching Logic

SPECTER should be supplied the same device name that appears in Intune for the
managed device record. If the local hostname, Azure AD device display name, and
Intune managed device name differ, use the Intune-managed name for best results.

---

## Data Collected

### 1. `compliance`

Compliance collection documents how Intune evaluated the device against assigned
policies. Depending on tenant configuration, data may include:

- Device compliance state
- Policy evaluation status
- Non-compliance reasons
- Last check-in timestamps
- Management ownership or enrollment state

This can be important when determining whether the endpoint was falling out of
policy before or during the incident window.

### 2. `installed_apps`

Installed application collection retrieves detected software inventory associated
with the managed device. Typical data includes:

- Application name
- Version
- Publisher
- Detection timestamps, where available

This provides a management-plane view of software presence that can be compared
against host-based artifact analysis.

### 3. `config_profiles`

Configuration profile collection records device configurations applied through
Intune, which may include:

- Security baseline assignments
- Device restriction profiles
- Endpoint protection settings
- Other applied configuration objects relevant to the managed device

This helps explain policy-driven settings that may affect encryption, telemetry,
or access controls on the endpoint.

### 4. `recovery_keys`

When enabled and permitted, SPECTER can retrieve **BitLocker recovery keys**
associated with the device through Microsoft Graph.

Use this capability only where legally authorized and operationally necessary.
Recovery key access is sensitive and should be tightly controlled.

### 5. `audit_logs`

Audit log collection retrieves device-related administrative actions from the
Intune or device-management audit stream, such as:

- Remote administrative changes
- Policy assignments or removals
- Device actions initiated by administrators
- Other relevant management events linked to the device

This can help establish what was done to the endpoint from the management plane
and when.

---

## Output Format

Each enabled collection area is written as a separate JSON file. SPECTER
preserves the Graph API response structure where practical so the resulting
artifacts remain traceable to the upstream service.

Typical output files include:

- `compliance.json`
- `installed_apps.json`
- `config_profiles.json`
- `recovery_keys.json`
- `audit_logs.json`

Because Microsoft Graph commonly returns paged JSON objects with `value`
collections and OData metadata, examiners should expect the files to reflect the
native Graph response format rather than a flattened export.

---

## Scoping and Privacy

This integration is intended to collect **device-level management data only**.
It is not intended to access user mailbox content, SharePoint files, OneDrive
documents, Teams chats, or unrelated personal content.

When configured as described in this guide, the focus is limited to:

- Managed device records
- Device compliance information
- Device application inventory
- Device configuration assignments
- Device recovery keys, when explicitly enabled
- Device-related audit events

This keeps the integration aligned with endpoint acquisition and minimizes access
to unrelated user data.

---

## YAML Configuration Example

```yaml
api_integration:
  provider: intune
  tenant_id: "11111111-2222-3333-4444-555555555555"
  client_id: "66666666-7777-8888-9999-aaaaaaaaaaaa"
  client_secret: ${INTUNE_CLIENT_SECRET}
  device_name: WS-02341
  options:
    compliance: true
    installed_apps: true
    config_profiles: true
    recovery_keys: false
    audit_logs: true
```

For secure deployments, store the client secret in your secret-management
workflow and inject it at runtime rather than hard-coding it in a broadly shared
configuration file.

---

## Troubleshooting

### `AADSTS700016`

Cause:
- Wrong `tenant_id`
- Wrong `client_id`
- App registration does not exist in the specified tenant

Resolution:
- Verify the tenant and application identifiers from the Azure Portal
- Confirm the app registration exists in the target tenant
- Ensure the token request is directed to the correct tenant authority

### 403 Forbidden

Cause:
- Missing Graph API permissions
- Admin consent has not been granted
- Optional permission such as `BitlockerKey.Read.All` not approved

Resolution:
- Review **API permissions** in the app registration
- Confirm **Grant admin consent** has been completed
- Remove unneeded collection options if the tenant will not authorize them

### Device Not Found

Cause:
- `device_name` does not exactly match the Intune managed device name

Resolution:
- Check the precise device name in Intune
- Confirm the device is enrolled and present in the tenant
- Verify naming differences between local hostname and Intune-managed display name

### Token Expiry or Authentication Failure

Cause:
- Expired or deleted client secret
- Incorrect secret value

Resolution:
- Generate a new client secret if needed
- Update the SPECTER configuration or secret store
- Re-run the collection with the corrected credential

---

## Related Documentation

- [Evidence Collection Guide](../EVIDENCE-GUIDE.md)
- [Working with SPECTER Evidence Artifacts](../WORKING-WITH-EVIDENCE.md)
- [SPECTER Technical Reference](../TECHNICAL-REFERENCE.md)
