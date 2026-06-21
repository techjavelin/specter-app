# SPECTER Evidence Collection Guide

## Forensic Best Practices, Chain of Custody, and Legal Considerations

**Document Version:** 1.0.0
**Last Updated:** 2026-06-21

---

## IMPORTANT LEGAL NOTICE

**This document provides general guidance based on publicly available forensic
standards and is not legal advice.** The information herein references published
standards from NIST, ISO/IEC, and SWGDE. It does not constitute legal counsel,
and no attorney-client relationship is created by reading or following this guide.

**You must consult qualified legal counsel in your jurisdiction before using
SPECTER or any forensic tool for evidence collection.** Laws governing digital
evidence collection, preservation, and admissibility vary by jurisdiction, case
type, and regulatory framework. Failure to comply with applicable laws may
result in evidence being deemed inadmissible or may expose you to legal liability.

**Tech Javelin, Ltd., its officers, employees, contractors, and affiliates accept
no responsibility or liability for any legal consequences, adverse rulings,
evidence challenges, regulatory penalties, or other outcomes that may arise from
the use of this software or the guidance contained in this document.** Users
assume all risk associated with evidence collection activities performed using
SPECTER.

---

## Referenced Standards

This guide references the following publicly available standards. Users should
obtain and review the current versions of these documents:

| Standard | Title | Publisher |
|----------|-------|-----------|
| NIST SP 800-86 | Guide to Integrating Forensic Techniques into Incident Response | National Institute of Standards and Technology (2006) |
| ISO/IEC 27037:2012 | Guidelines for identification, collection, acquisition and preservation of digital evidence | International Organization for Standardization |
| SWGDE 18-F-002 | Best Practices for Digital Evidence Collection | Scientific Working Group on Digital Evidence (2025) |
| SWGDE 17-F-002 | Best Practices for Computer Forensic Acquisitions | Scientific Working Group on Digital Evidence (2025) |
| FRE 901 | Requirement of Authentication or Identification | Federal Rules of Evidence (U.S.) |
| FRE 902(13)(14) | Self-Authenticating Certified Records and Copied Data | Federal Rules of Evidence (U.S.) |

**Note:** Jurisdictions outside the United States may have different evidentiary
standards. The Federal Rules of Evidence are cited as a reference framework.
Consult the applicable rules of evidence in your jurisdiction.

---

## 1. Pre-Collection Requirements

Before running SPECTER, the following documentation and preparation steps are
required per NIST SP 800-86 Section 4 and ISO/IEC 27037:2012.

### 1.1 Legal Authorization

- **Obtain written legal authorization** before collecting evidence. This may
  include a court order, warrant, consent form, or organizational policy
  authorization depending on your jurisdiction and context.
- **Document the legal basis** for the collection (e.g., warrant number, consent
  form reference, incident ticket, or HR authorization reference).
- **Confirm scope**: Ensure the authorization covers the specific devices,
  data types, and storage media you intend to image.

### 1.2 Pre-Collection Documentation

Per NIST SP 800-86 and SWGDE 18-F-002, document the following before imaging:

| Item | Description |
|------|-------------|
| Case/Incident ID | Unique identifier for the matter |
| Date and time (with timezone) | When documentation was initiated |
| Examiner name and credentials | Person performing the acquisition |
| Authorization reference | Warrant, consent form, or policy citation |
| Device description | Make, model, serial number, condition |
| Device location | Physical location where device was found |
| Device state | Power on/off, screen locked/unlocked, connected peripherals |
| Photograph(s) | Photos of device, screen state, connected cables, and physical condition |
| Witnesses present | Names and roles of persons present during collection |

### 1.3 Scene Documentation

- **Photograph the device** in its found state before touching it.
- **Document all connected peripherals**, network cables, and external media.
- **Note the power state** -- if the device is powered on, volatile memory may
  contain evidence. Decide whether to perform live acquisition (memory capture)
  before powering down.

---

## 2. Physical Device Handling and Labeling

Per ISO/IEC 27037:2012 and SWGDE 18-F-002, physical evidence must be labeled,
packaged, and tracked.

### 2.1 Device Labeling

Every device and storage medium must be labeled with:

- **Evidence tag/label number** (unique, sequential)
- **Case/Incident ID**
- **Date and time of seizure/collection**
- **Description** (e.g., "Dell Latitude 5520, S/N: ABC123, 512GB NVMe")
- **Collected by** (examiner name)
- **Tamper-evident seal** (evidence tape or tamper-evident bag with unique seal number)

### 2.2 External/Target Drives

The drive used to store SPECTER output must also be labeled:

- **Drive serial number** (SPECTER records this in the manifest)
- **Case association** (which case this drive is assigned to)
- **"EVIDENCE -- DO NOT FORMAT"** or equivalent marking
- **Write-protection status** after imaging is complete

### 2.3 Packaging

- Use **anti-static bags** for hard drives and SSDs.
- Use **tamper-evident evidence bags** with unique serial numbers.
- Store devices away from **magnetic fields, extreme temperatures, and moisture**.
- Per ISO/IEC 27037:2012, protect evidence from **physical, environmental, and
  electromagnetic hazards**.

---

## 3. Chain of Custody

Chain of custody documentation demonstrates unbroken control of evidence from
collection through presentation. Per NIST SP 800-86 Section 4 and FRE 901(b)(1),
a documented chain of custody supports authentication of evidence.

### 3.1 Chain of Custody Record

Maintain a written chain of custody log for each evidence item. Each entry must
include:

| Field | Description |
|-------|-------------|
| Evidence item ID | Matches the physical label |
| Date and time | When the transfer occurred |
| Released by | Name and signature of person releasing |
| Received by | Name and signature of person receiving |
| Purpose | Reason for transfer (e.g., "forensic imaging", "storage", "legal review") |
| Location | Where the evidence was transferred to |
| Condition | Condition of evidence at time of transfer |

### 3.2 Continuity Requirements

- **Every transfer** of physical or digital evidence must be logged.
- **Secure storage** must be access-controlled with logged entry/exit.
- **Gaps in custody** (periods where evidence was unaccounted for) may be grounds
  for challenge under FRE 901.

### 3.3 Digital Chain of Custody

SPECTER's manifest (`specter-manifest.json`) provides a digital chain of custody
record that includes:

- Examiner identification
- Device serial number and hostname
- Acquisition timestamps (start, end, sealed)
- Cryptographic hashes (SHA-256) of all collected artifacts
- HMAC-SHA256 integrity seal across the complete evidence package
- Tool versions used for acquisition
- Verification status of each disk image

This manifest, combined with physical chain of custody documentation, supports
authentication under FRE 901(b)(9) (process or system evidence) and FRE 902(14)
(certified data copied from an electronic device).

---

## 4. Integrity Verification and Digital Signing

### 4.1 Hash Verification

SPECTER generates SHA-256 hashes for all collected artifacts and records them in
the manifest. Per NIST SP 800-86, hash verification demonstrates that evidence
has not been altered since collection.

- **At acquisition**: ewfacquire generates and verifies E01 image hashes.
- **At sealing**: SPECTER records the final hash of every artifact.
- **At verification**: Re-hash artifacts and compare against manifest values.

### 4.2 HMAC Integrity Seal

SPECTER generates an HMAC-SHA256 seal derived from deterministic case metadata.
This seal allows detection of any post-collection modification to the manifest
or its referenced artifacts.

### 4.3 Digital Signing (Recommended)

For additional evidentiary weight, consider digitally signing the sealed manifest
and/or the evidence archive:

- **Code signing certificates** (e.g., X.509) from a trusted Certificate
  Authority provide non-repudiation.
- **Timestamping** via an RFC 3161 Time Stamping Authority (TSA) establishes
  when the signature was created, independent of local system time.
- **PGP/GPG signing** with a published public key provides an alternative
  mechanism for integrity verification.

Per FRE 902(13) and 902(14), a certification from a qualified person describing
the electronic process used to generate or copy data may render the evidence
self-authenticating. Digital signatures support this certification.

**Implementation example:**

```bash
# Sign the sealed manifest with GPG
gpg --detach-sign --armor specter-manifest.json

# Timestamp with a public TSA (example using OpenSSL)
openssl ts -query -data specter-manifest.json -no_nonce -sha256 -out request.tsq
curl -H "Content-Type: application/timestamp-query" \
  --data-binary @request.tsq https://freetsa.org/tsr -o response.tsr
```

**Note:** The specific signing mechanism and acceptable Certificate Authorities
will depend on your jurisdiction's requirements. Consult legal counsel.

---

## 5. Evidence Preservation and Cold Storage

### 5.1 Storage Requirements

Per NIST SP 800-86 and ISO/IEC 27037:2012, preserved evidence must be:

- Stored in a **secure, access-controlled location**
- Protected from **environmental hazards** (temperature, humidity, electromagnetic interference)
- **Air-gapped** from networks when possible to prevent remote tampering
- **Encrypted at rest** using strong cryptographic standards (AES-256 or equivalent)

### 5.2 Storage Media

| Medium | Use Case | Lifespan (estimated) |
|--------|----------|---------------------|
| LTO magnetic tape | Large-volume archival | 15-30 years (manufacturer-rated) |
| Archival-grade optical (M-DISC) | Small to medium evidence packages | 100+ years (manufacturer-rated) |
| Enterprise SSD/HDD | Active case storage | 3-5 years (requires periodic verification) |
| WORM cloud storage (e.g., S3 Object Lock, Azure Immutable Blob) | Cloud-based preservation | Duration of service agreement |

**Note:** Estimated lifespans are manufacturer claims. Actual durability depends
on storage conditions. Periodic integrity verification is required regardless of
medium.

### 5.3 Integrity Monitoring

- **Verify hashes** against the manifest at minimum every 12 months.
- **Migrate to new media** before the current medium reaches end of rated life.
- **Document all verification and migration events** in the chain of custody log.
- **Retain the original manifest** alongside migrated copies.

### 5.4 Retention Periods

Retention periods vary by jurisdiction, case type, and regulatory framework.
The following are general reference points only:

| Context | Typical Retention | Authority |
|---------|-------------------|-----------|
| Active litigation | Until final resolution of all appeals | Applicable rules of civil/criminal procedure |
| Criminal cases (felony) | Per jurisdiction statute (commonly 7-10+ years) | State/federal law |
| Criminal cases (homicide) | Indefinite (commonly life of case) | State/federal law |
| Regulatory compliance (financial) | 5-7 years minimum | SOX, GLBA, applicable regulations |
| Regulatory compliance (healthcare) | 6 years minimum | HIPAA (45 CFR 164.530) |
| Internal investigations | Per organizational retention policy | Organizational policy |

**You must determine the applicable retention period for your jurisdiction and
case type. Consult legal counsel before disposing of any evidence.**

Premature destruction of evidence may constitute **spoliation**, which can result
in adverse inference instructions, sanctions, or criminal penalties depending on
jurisdiction.

---

## 6. Cloud Storage: Tenancy, Permissions, Logging, and Retention

When SPECTER uploads evidence to cloud storage (AWS S3, Google Cloud Storage,
Azure Blob Storage), the following controls should be implemented.

### 6.1 Tenancy Isolation

- Use a **dedicated cloud account or subscription** for evidence storage,
  separate from production workloads.
- If a shared account is required, use a **dedicated bucket/container** with
  strict access boundaries.
- Enable **versioning** on the storage bucket/container to prevent accidental
  overwrites.

### 6.2 Access Controls

- Apply the **principle of least privilege**: only authorized examiners and
  legal hold administrators should have access.
- Use **IAM policies** (not access keys where possible) scoped to the specific
  evidence bucket/container.
- Require **multi-factor authentication (MFA)** for all accounts with evidence
  access.
- **Disable public access** at the bucket/container level.
- Use **service-specific access logging** to record all access attempts.

### 6.3 Immutability and Write Protection

| Provider | Immutability Mechanism | Documentation |
|----------|----------------------|---------------|
| AWS S3 | Object Lock (Compliance mode) | docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html |
| Google Cloud Storage | Bucket Lock (retention policy) | cloud.google.com/storage/docs/bucket-lock |
| Azure Blob Storage | Immutable storage (legal hold or time-based retention) | learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview |

- Enable **compliance-mode object lock** or equivalent. Compliance mode prevents
  deletion or modification by any user, including root/owner accounts, until the
  retention period expires.
- Set retention periods that align with your legal retention requirements.

### 6.4 Access Logging and Auditing

- **AWS**: Enable S3 Server Access Logging and/or CloudTrail data events for the
  evidence bucket.
- **GCS**: Enable Cloud Audit Logs (Data Access logs) for the evidence bucket.
- **Azure**: Enable Storage Analytics logging and/or Azure Monitor diagnostic
  settings for the evidence container.

Log the following at minimum:

- Who accessed the evidence (identity/principal)
- When (timestamp)
- What operation was performed (read, write, delete, list)
- Source IP address
- Success or failure of the operation

**Retain access logs for at least the same duration as the evidence itself.**

### 6.5 Encryption

- Enable **server-side encryption** (SSE) with customer-managed keys (CMK) where
  possible (AWS SSE-KMS, GCS CMEK, Azure CMK).
- Enable **encryption in transit** (TLS 1.2+). SPECTER uses HTTPS for all uploads.
- Store encryption key metadata in the chain of custody documentation.

---

## 7. Operational Checklist

The following checklist summarizes the key steps. This is a reference tool, not
a substitute for training or legal counsel.

### Pre-Collection

- [ ] Legal authorization obtained and documented
- [ ] Case/incident ID assigned
- [ ] Examiner credentials and authorization verified
- [ ] Device photographed in found state
- [ ] Device state documented (power, peripherals, network)
- [ ] Witnesses identified and documented
- [ ] Target/evidence drive prepared and labeled
- [ ] Chain of custody form initiated

### During Collection (SPECTER Operation)

- [ ] SPECTER run from external/portable media (not installed on target)
- [ ] Memory capture performed first (if device is powered on)
- [ ] Disk acquisition completed with integrity verification
- [ ] SPECTER manifest generated and sealed
- [ ] No additional software installed on target device

### Post-Collection

- [ ] All evidence items labeled with evidence tags
- [ ] Chain of custody entries completed for all transfers
- [ ] Evidence packaged in tamper-evident containers
- [ ] Digital signatures applied to manifest (recommended)
- [ ] Evidence transported to secure storage
- [ ] Cloud upload verified (if applicable)
- [ ] Immutability/object lock enabled on cloud storage
- [ ] Access controls verified on storage location
- [ ] Retention period set per legal requirements

---

## 8. Admissibility Considerations

The following factors are commonly evaluated when digital evidence is challenged.
These are based on FRE 901, FRE 902(13)(14), and general forensic practice as
documented in NIST SP 800-86.

| Factor | What Courts Examine | How SPECTER Supports |
|--------|--------------------|--------------------|
| Authenticity | Is this evidence what it claims to be? | SHA-256 hashes, HMAC seal, examiner metadata |
| Integrity | Has the evidence been altered? | Hash verification, HMAC integrity seal, E01 internal checksums |
| Chain of custody | Was evidence continuously controlled? | Manifest timestamps, examiner ID, physical CoC documentation |
| Reliability of process | Was the collection method sound? | Forensically validated tool (ewfacquire), no target modification |
| Completeness | Is all relevant evidence accounted for? | Manifest itemizes all collected artifacts |

**SPECTER provides technical controls that support admissibility. However,
admissibility ultimately depends on compliance with applicable procedural rules,
proper physical handling, and adequate documentation maintained by the
examiner.** No software tool alone can guarantee admissibility.

---

## 9. Limitations and Disclaimers

1. **SPECTER is a forensic acquisition tool, not a complete evidence management
   system.** Physical chain of custody, witness documentation, and legal
   authorization must be maintained independently by the examiner.

2. **This guide references standards current as of its publication date.**
   Standards are periodically revised. Verify you are referencing current
   versions from the issuing organizations (NIST, ISO, SWGDE).

3. **Jurisdictional variation:** This guide references U.S. Federal Rules of
   Evidence. Other jurisdictions (state, international) have different
   evidentiary frameworks. Always verify against applicable local law.

4. **No warranty of admissibility:** Neither this guide nor SPECTER guarantees
   that evidence will be admitted in any proceeding. Admissibility is
   determined by courts based on the totality of circumstances.

5. **Tech Javelin, Ltd. provides this software and documentation "as is"
   without warranty of any kind.** See the LICENSE file for complete terms.

6. **Seek qualified legal counsel** before relying on any information in this
   document for active legal proceedings.

---

## References

- National Institute of Standards and Technology. (2006). *Guide to Integrating
  Forensic Techniques into Incident Response* (SP 800-86).
  https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-86.pdf

- International Organization for Standardization. (2012). *Information technology
  -- Security techniques -- Guidelines for identification, collection,
  acquisition and preservation of digital evidence* (ISO/IEC 27037:2012).

- Scientific Working Group on Digital Evidence. (2025). *Best Practices for
  Digital Evidence Collection* (18-F-002-2.0).
  https://www.swgde.org/documents/published-complete-listing/

- Scientific Working Group on Digital Evidence. (2025). *Best Practices for
  Computer Forensic Acquisitions* (17-F-002-2.1).
  https://www.swgde.org/documents/published-complete-listing/

- Federal Rules of Evidence, Rules 901, 902.
  https://www.law.cornell.edu/rules/fre

---

*Copyright 2026 Tech Javelin, Ltd. All rights reserved.*
*This document is provided for informational purposes only and does not
constitute legal advice.*
