# Cloud Targets Guide

## Configuring Remote Evidence Output Destinations

**Document Version:** 1.0.0
**Last Updated:** 2026-06-23

---

## Overview

SPECTER can write evidence output to multiple destination types so collections
can be preserved off-host during or immediately after acquisition. Supported
deployment patterns typically include:

- Local filesystem destinations
- Network shares
- AWS S3
- Azure Blob Storage
- Google Cloud Storage

Using remote object storage is useful for rapid preservation, geographic
redundancy, and centralized evidence handling. In many environments, the best
practice is to store evidence both locally and remotely.

---

## Multi-Destination Output

SPECTER can be configured to write to both a **local destination** and a
**remote destination** simultaneously.

This model is often preferred because it provides:

- Immediate local access for the examiner or responder
- Redundant remote preservation
- Reduced risk if one destination becomes unavailable

Use local storage for near-term access and remote storage for durable retention
and centralized chain-of-custody handling.

---

## AWS S3

### Minimal IAM Permissions

Grant only the permissions required to upload evidence into the intended
bucket/prefix.

At the S3 API level, the minimal upload path should allow:

- `s3:PutObject`
- multipart upload create
- multipart upload complete
- multipart upload abort

In AWS IAM, multipart create and complete are typically authorized through
`s3:PutObject`, while cancellation requires `s3:AbortMultipartUpload`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:AbortMultipartUpload"
      ],
      "Resource": "arn:aws:s3:::specter-evidence/case-uploads/*"
    }
  ]
}
```

If your environment requires additional multipart-related permissions based on
SDK, gateway, or bucket-policy behavior, add only the narrowly scoped actions
required by your implementation review.

### Bucket Setup

Recommended bucket settings:

- **Versioning enabled**
- **Server-side encryption** using AES-256 or AWS KMS
- **Lifecycle rule** to clean up incomplete multipart uploads
- Access logging or CloudTrail data event logging enabled for audit purposes

### Configuration Example

```yaml
output:
  local_path: /evidence/local-copy
  remote:
    provider: s3
    bucket: specter-evidence
    prefix: case-uploads/INC-2026-0042/
    region: us-east-1
    encryption: AES256
```

### Cross-Region Considerations

Cross-region uploads are supported, but increased latency can reduce throughput
and extend acquisition completion time. Where possible, use a bucket in the same
or a nearby region to the target host.

---

## Azure Blob Storage

### Access Methods

SPECTER deployments commonly authenticate to Azure Blob Storage using one of the
following methods:

- **SAS token** with a time-limited scope
- **Storage account key**
- **Managed identity** when SPECTER runs on an Azure VM or other Azure-hosted
  compute environment with appropriate role assignment

Prefer the least-privileged method that fits the deployment model. For temporary
collection workflows, SAS tokens are often the best operational fit.

### Container Setup

Recommended container settings:

- **Hot** access tier for active uploads
- Encryption at rest enabled
- Access logging enabled
- Restrict public access

### Configuration Example

```yaml
output:
  remote:
    provider: azure_blob
    account_name: specterevidence
    container: incident-uploads
    path_prefix: INC-2026-0042/
    auth:
      method: sas
      sas_token: ${AZURE_BLOB_SAS}
```

If using a storage account key or managed identity, replace the `auth` section
with the appropriate credential method for your environment.

---

## Google Cloud Storage

### Service Account Permissions

Create a service account with permission to upload objects to the intended
bucket. At minimum, grant:

- `storage.objects.create`

Scope the permission to the specific bucket used for SPECTER evidence whenever
possible.

### Bucket Configuration

Recommended settings:

- **Standard** storage class
- **Uniform bucket-level access**
- Provider-managed encryption by default, with **CMEK** optional where required
  by policy
- Audit logging enabled

### Configuration Example

```yaml
output:
  remote:
    provider: gcs
    bucket: specter-evidence
    prefix: INC-2026-0042/
    credentials_file: /etc/specter/gcs-service-account.json
```

Protect the service account credential file with strict filesystem permissions if
it is used outside a managed identity or workload identity model.

---

## Network Considerations

### Bandwidth Throttling

Where remote links are constrained, use:

- `max_bandwidth_mbps`

This allows uploads to be rate-limited to reduce operational impact on the
target environment or WAN links.

### Resumable Uploads

SPECTER supports segmented output behavior so large evidence components can be
uploaded independently. This improves resilience because interrupted transfers do
not require the entire collection set to be retransmitted from the beginning.

### Integrity Verification After Upload

After upload completes, perform post-transfer integrity verification by comparing
the local SHA-256 values in the SPECTER manifest against the hashes of the
objects stored remotely.

Where the cloud provider exposes object checksums or where a secondary verifier
is available, compare those values and record the results in the case notes.

---

## Security Considerations

SPECTER remote uploads should be configured with the same evidentiary care as any
other transport path.

Recommended controls:

- **TLS 1.2 or later** for all transport connections
- **At-rest encryption** using provider-managed or customer-managed keys
- **Access logging** to maintain an audit trail of evidence writes and reads
- **Retention locks** or **legal hold** where supported and appropriate for
  evidence preservation requirements

Restrict credentials so they can upload to the intended case prefix without broad
enumeration or read access unless that access is explicitly required by the
workflow.

---

## Related Documentation

- [MDM Deployment Guide](./MDM-DEPLOYMENT.md)
- [Evidence Collection Guide](../EVIDENCE-GUIDE.md)
- [Working with SPECTER Evidence Artifacts](../WORKING-WITH-EVIDENCE.md)
- [SPECTER Technical Reference](../TECHNICAL-REFERENCE.md)
