# SPECTER

```
  ____  ____  _____  ____  _____  _____  ____
 / ___||  _ \| ____|| ___||_   _|| ____||  _ \
 \___ \| |_) |  _|  | |     | |  |  _|  | |_) |
  ___) |  __/| |___ | |___  | |  | |___ |  _ <
 |____/|_|   |_____| \____| |_|  |_____||_| \_\
                 Forensic Acquisition Engine
```

**by Tech Javelin**

[![Release](https://img.shields.io/github/v/release/techjavelin/specter-app?label=release)](https://github.com/techjavelin/specter-app/releases)
[![License](https://img.shields.io/badge/license-proprietary-red.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-blue.svg)]()

> *Clone. Verify. Deliver.*

SPECTER is a cross-platform forensic disk acquisition toolkit written in Go. It produces court-admissible E01 disk images with HMAC-sealed chain-of-custody manifests, captures volatile state and system triage data, and supports streaming upload to cloud storage (AWS S3, Google Cloud Storage, Azure Blob).

---

## Why SPECTER?

A specter is a ghost -- an exact but intangible echo of something that once existed. That is precisely what forensic disk imaging does: it creates a ghost image of a device, a perfect bit-for-bit copy that preserves the original state without altering it. The image is the specter of the machine.

The deliberate misspelling -- SPECTER, not SPECTRE -- is intentional. This is not a Bond villain or a CVE. It is a nod to the American English spelling and a quiet declaration that this tool is its own thing. It also serves as a hat tip to the era that inspired it: the rambunctious, irreverent, ASCII-art-in-your-MOTD hacker culture of the 1990s, when tools had personality, names meant something, and nobody spelled anything the way you expected them to.

SPECTER creates ghosts. That is what it does. That is why it is called what it is called.

---

## Features

| Capability | Description |
|------------|-------------|
| **Disk Imaging** | E01 format via ewfacquire with configurable segment sizes |
| **Memory Capture** | WinPmem (Windows), osxpmem (macOS/Intel), LiME (Linux) |
| **Volatile Collection** | Running processes, network connections, DNS cache, ARP table |
| **Triage Collection** | Registry hives, event logs, installed software, system logs |
| **Interactive TUI** | Bubble Tea wizard with step-by-step guided acquisition |
| **Headless Mode** | YAML config-driven for MDM/scripted deployment |
| **HMAC Seal** | Cryptographic chain-of-custody manifest (SHA-256 + HMAC) |
| **Cloud Upload** | Streaming segment upload to S3, GCS, or Azure Blob |
| **Verify Command** | Independent integrity verification of sealed evidence packets |
| **Cross-Platform** | Single binary for Windows, macOS (Intel/ARM), Linux |

## Installation

Download the latest release binary for your platform from [GitHub Releases](https://github.com/techjavelin/specter-app/releases).

Extract the archive and place the binary in your PATH or run it directly from a USB drive or network share.

```bash
# macOS/Linux -- make executable
chmod +x specter
sudo mv specter /usr/local/bin/

# Windows -- place in PATH or run from USB/network share
```

**Platform notes:**

- **macOS**: You may need to allow the binary in System Settings > Privacy & Security after the first run. All tools are bundled.
- **Linux**: Fully self-contained. All tools are bundled as statically-linked binaries in the `tools/` directory.
- **Windows**: Run from an elevated (Administrator) command prompt or PowerShell. All tools are bundled.

## Quick Start

### Interactive Mode (TUI)

```bash
sudo specter acquire
```

Launches the guided wizard:
1. Enter examiner name and case description
2. Select target disk from detected devices
3. Choose output directory
4. Configure segment size (default: 4GB)
5. Select acquisition phases (memory, volatile, triage, disk)
6. Review and confirm

### Headless Mode

```bash
sudo specter acquire --headless --config specter.yaml
```

For scripted/MDM deployment. All parameters from YAML config:

```yaml
examiner: "Chris Schmidt"
description: "IR-2026-0042 Workstation Acquisition"
output_dir: "/mnt/evidence"
source_disk: "/dev/sda"
segment_size: "4G"
phases:
  - memory
  - volatile
  - triage
  - disk
upload:
  provider: s3
  bucket: evidence-bucket
  region: us-east-1
```

### MDM and Scripted Deployment

SPECTER's headless mode is designed for remote and automated deployment. The general workflow is:

1. **Stage** the SPECTER binary and a YAML config file on the target endpoint (via MDM file distribution, network share, or download from your release URL).
2. **Execute** `specter acquire --headless --config specter.yaml` remotely.
3. **Stream** evidence directly to cloud storage -- no need to retrieve large files from the endpoint afterward.
4. **Clean up** the binary and config after acquisition completes.

#### Scenario: Remote IR Triage

An analyst receives an alert for a compromised endpoint. Without needing physical access, they push SPECTER to the machine and trigger a headless acquisition with cloud upload. Evidence streams directly to the team's S3 bucket for immediate analysis:

```yaml
examiner: "SOC Analyst"
description: "ALERT-7291 endpoint compromise"
phases:
  - memory
  - volatile
  - triage
output:
  remote: s3://ir-evidence/cases/ALERT-7291/
credentials:
  aws_access_key_id: ${AWS_ACCESS_KEY_ID}
  aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  aws_region: us-east-1
```

#### Scenario: Fleet-Wide Collection

Legal hold or compliance review requires volatile and triage data from all endpoints in a business unit. Push a minimal config -- no disk imaging -- to collect state quickly with minimal endpoint impact:

```yaml
examiner: "Compliance Team"
description: "Legal Hold LH-2026-003"
phases:
  - volatile
  - triage
output:
  remote: gs://compliance-evidence/LH-2026-003/${HOSTNAME}/
credentials:
  google_credentials_file: /opt/specter/gcs-key.json
upload:
  max_bandwidth_mbps: 25
```

#### JumpCloud

Use JumpCloud Commands to download and execute SPECTER on Linux or macOS endpoints:

```bash
# JumpCloud Command (Linux/macOS)
curl -sL https://github.com/techjavelin/specter-app/releases/latest/download/specter_VERSION_linux_amd64.zip -o /tmp/specter.zip
unzip -o /tmp/specter.zip -d /tmp/specter
cat > /tmp/specter/specter.yaml << 'EOF'
examiner: "SOC Team"
phases:
  - volatile
  - triage
output:
  remote: s3://evidence-bucket/jumpcloud/${HOSTNAME}/
credentials:
  aws_access_key_id: ${AWS_ACCESS_KEY_ID}
  aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  aws_region: us-east-1
EOF
sudo /tmp/specter/specter acquire --headless --config /tmp/specter/specter.yaml
rm -rf /tmp/specter /tmp/specter.zip
```

Replace `VERSION` with the desired release version (e.g., `1.0.1`).

#### Microsoft Intune

Use Intune Remediation Scripts to deploy SPECTER on Windows endpoints:

```powershell
# Intune Remediation Script (Windows)
$specterUrl = "https://github.com/techjavelin/specter-app/releases/latest/download/specter_VERSION_windows_amd64.zip"
$workDir = "$env:TEMP\specter"
New-Item -ItemType Directory -Force -Path $workDir | Out-Null
Invoke-WebRequest -Uri $specterUrl -OutFile "$workDir\specter.zip"
Expand-Archive -Path "$workDir\specter.zip" -DestinationPath $workDir -Force

# Write config
@"
examiner: "IR Automation"
phases:
  - memory
  - volatile
  - triage
output:
  remote: az://evidence/intune/$env:COMPUTERNAME/
credentials:
  azure_storage_account: myevidencestorage
  azure_storage_key: $env:AZURE_STORAGE_KEY
"@ | Set-Content "$workDir\specter.yaml"

Start-Process -FilePath "$workDir\specter.exe" -ArgumentList "acquire","--headless","--config","$workDir\specter.yaml" -Wait -NoNewWindow
Remove-Item -Recurse -Force $workDir
```

Replace `VERSION` with the desired release version.

#### Jamf Pro

Use a Jamf Pro policy with a script payload to deploy SPECTER on macOS endpoints:

```bash
#!/bin/bash
# Jamf Pro Script Payload (macOS)
WORK_DIR=$(mktemp -d)
curl -sL https://github.com/techjavelin/specter-app/releases/latest/download/specter_VERSION_darwin_arm64.zip -o "$WORK_DIR/specter.zip"
unzip -o "$WORK_DIR/specter.zip" -d "$WORK_DIR/specter"
chmod +x "$WORK_DIR/specter/specter"

cat > "$WORK_DIR/specter/specter.yaml" << 'EOF'
examiner: "Security Operations"
phases:
  - volatile
  - triage
output:
  remote: gs://evidence-bucket/jamf/$(hostname)/
credentials:
  google_credentials_file: /Library/Application Support/specter/gcs-key.json
EOF

"$WORK_DIR/specter/specter" acquire --headless --config "$WORK_DIR/specter/specter.yaml"
rm -rf "$WORK_DIR"
```

Replace `VERSION` with the desired release version (e.g., `1.0.1`). For Apple Silicon Macs, use `darwin_arm64`; for Intel Macs, use `darwin_amd64`.

Pre-deploy the GCS service account key to endpoints via Jamf's file distribution or a configuration profile before running the acquisition script.

#### Best Practices for MDM Deployment

- **Always use cloud upload** (`output.remote`) so evidence streams directly to your storage bucket. This avoids filling local disk on the endpoint.
- **Use environment variables** (`${ENV_VAR}`) for credentials in config files. Never hardcode secrets.
- **Skip disk imaging for fleet-wide triage.** Use `phases: [volatile, triage]` to minimize endpoint impact and acquisition time.
- **Set `segment_preset: cloud`** when using cloud upload for optimal upload granularity (2 GB segments).
- **Manage bandwidth.** Set `max_bandwidth_mbps` to a reasonable value to avoid saturating the endpoint's network connection, especially during business hours.
- **Clean up after acquisition.** Remove the SPECTER binary, config file, and any temporary files from the endpoint once the acquisition completes.
- **Use unique prefixes per endpoint** (hostname, asset tag) in the cloud storage path to organize evidence and prevent collisions during fleet-wide collection.

### Verify Evidence

```bash
specter verify ./HOSTNAME_20260620_143800
```

Recalculates HMAC and file hashes, reports pass/fail for each file.

### Seal Evidence (standalone)

```bash
specter seal ./HOSTNAME_20260620_143800 --examiner "Chris Schmidt"
```

Generates `specter-manifest.json` with HMAC integrity seal. Useful when acquisition phases were run separately.

### Import Existing Image

```bash
specter import ./existing-image.E01 --examiner "Chris Schmidt"
```

Creates a SPECTER evidence packet from a pre-existing E01 image.

## Commands

| Command | Description |
|---------|-------------|
| `specter acquire` | Run acquisition (interactive TUI by default) |
| `specter acquire --headless` | Run acquisition from YAML config |
| `specter acquire --resume <path>` | Resume an interrupted acquisition |
| `specter verify <path>` | Verify integrity of sealed evidence |
| `specter seal <path>` | Generate chain-of-custody seal |
| `specter import <e01>` | Import existing E01 into SPECTER format |
| `specter version` | Print version, commit, and build date |

## Resuming an Interrupted Acquisition

If SPECTER is interrupted during acquisition -- whether by power failure, network drop, or forced quit -- the state file (`specter-state.json`) preserves progress for each phase.

To resume, run:

```bash
specter acquire --resume /path/to/output
```

**How resume works:**

- **Completed phases are skipped.** If memory and volatile collection finished before the interruption, they will not run again.
- **Interrupted phases restart from the beginning.** Partial results from an incomplete phase (such as a partially written E01 segment) are discarded, and the phase runs again in full.
- **SEAL runs automatically** after all phases complete, just as it would in a normal acquisition.

**Example:**

```bash
# Original acquisition was interrupted during disk imaging
sudo specter acquire --resume /mnt/evidence/HOSTNAME_20260620_143800
```

## Output Structure

```
HOSTNAME_20260620_143800/
├── specter-manifest.json    # HMAC-sealed chain-of-custody
├── specter-state.json       # Run state (resumable)
├── specter.log              # Full acquisition log
├── memory/
│   └── physmem.raw
├── volatile/
│   ├── processes.txt
│   ├── network_connections.txt
│   ├── dns_cache.txt
│   └── arp_table.txt
├── triage/
│   ├── registry/            # (Windows) SAM, SYSTEM, SOFTWARE, etc.
│   ├── eventlogs/           # (Windows) Security, System, Application
│   ├── installed_software.txt
│   └── system_logs/
└── disk/
    ├── disk0.E01
    ├── disk0.E02
    └── ...
```

## Chain of Custody (Seal)

The `specter-manifest.json` contains:
- Run metadata (version, platform, operator, timestamps)
- Target identity (hostname, OS, serial number)
- Per-file SHA-256 hashes
- HMAC signature over the complete manifest

### HMAC Key Derivation

The HMAC key is derived deterministically from acquisition metadata using the following components concatenated with pipe delimiters:

```
run_id|examiner|hostname|serial|sealed_at
```

This ensures the seal is reproducible given the same acquisition context, while remaining unique per run.

### Disk Image Hashing

Disk images (E01 files) use a `verified_by` field instead of re-hashing. Because `ewfacquire` and `ewfverify` already perform internal hash validation during the imaging process, SPECTER records the verification result from these tools rather than redundantly hashing potentially multi-terabyte image files.

### Manifest Validation

The manifest is validated against a JSON Schema before writing. The schema definition is available in the [specter-app repository](https://github.com/techjavelin/specter-app).

A full acquisition (all 4 phases) automatically seals on completion. Partial runs require explicit `--seal` flag or subsequent `specter seal` command.

## Cloud Upload

SPECTER streams E01 segments to cloud storage as they are written. Each provider uses its native SDK and supports its standard credential chain for authentication.

### AWS S3

**Setup:**

1. Create an S3 bucket:
   ```bash
   aws s3 mb s3://my-evidence-bucket --region us-east-1
   ```

2. Enable versioning (recommended for evidence integrity):
   ```bash
   aws s3api put-bucket-versioning --bucket my-evidence-bucket \
     --versioning-configuration Status=Enabled
   ```

3. Enable server-side encryption (AES-256 default or KMS):
   ```bash
   aws s3api put-bucket-encryption --bucket my-evidence-bucket \
     --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
   ```

4. Block public access:
   ```bash
   aws s3api put-public-access-block --bucket my-evidence-bucket \
     --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
   ```

5. Create IAM user/role with minimum permissions:
   - `s3:PutObject`, `s3:GetObject`, `s3:HeadObject` on `arn:aws:s3:::my-evidence-bucket/*`
   - `s3:ListBucket` on `arn:aws:s3:::my-evidence-bucket`

6. Generate access keys or configure IAM role for the acquisition workstation.

**SPECTER config:**

```yaml
output:
  remote: s3://my-evidence-bucket/cases/IR-2026-0042/
credentials:
  aws_access_key_id: ${AWS_ACCESS_KEY_ID}
  aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  aws_region: us-east-1
upload:
  concurrent_uploads: 3
  verify_after_upload: true
imaging:
  segment_preset: cloud   # 2 GB segments for upload granularity
```

If running on EC2 with an IAM role attached, omit the `credentials` section entirely. SPECTER uses the standard AWS credential chain (instance metadata, environment variables, shared credentials file).

### Google Cloud Storage

**Setup:**

1. Create a GCS bucket:
   ```bash
   gsutil mb -l us-east1 -c standard gs://my-evidence-bucket
   ```

2. Enable uniform bucket-level access:
   ```bash
   gsutil uniformbucketlevelaccess set on gs://my-evidence-bucket
   ```

3. Enable Object Versioning:
   ```bash
   gsutil versioning set on gs://my-evidence-bucket
   ```

4. Create a service account with minimum permissions:
   ```bash
   gcloud iam service-accounts create specter-uploader \
     --display-name="SPECTER Evidence Upload"
   ```

5. Grant `roles/storage.objectCreator` and `roles/storage.objectViewer` on the bucket:
   ```bash
   gsutil iam ch serviceAccount:specter-uploader@PROJECT.iam.gserviceaccount.com:roles/storage.objectCreator gs://my-evidence-bucket
   gsutil iam ch serviceAccount:specter-uploader@PROJECT.iam.gserviceaccount.com:roles/storage.objectViewer gs://my-evidence-bucket
   ```

6. Generate a JSON key:
   ```bash
   gcloud iam service-accounts keys create specter-gcs-key.json \
     --iam-account=specter-uploader@PROJECT.iam.gserviceaccount.com
   ```

**SPECTER config:**

```yaml
output:
  remote: gs://my-evidence-bucket/cases/IR-2026-0042/
credentials:
  google_credentials_file: /path/to/specter-gcs-key.json
upload:
  concurrent_uploads: 3
  verify_after_upload: true
imaging:
  segment_preset: cloud
```

If running on a GCE instance with a service account attached, omit the `credentials` section. SPECTER uses Application Default Credentials.

### Azure Blob Storage

**Setup:**

1. Create a storage account:
   ```bash
   az storage account create \
     --name myevidencestorage \
     --resource-group forensics-rg \
     --location eastus \
     --sku Standard_LRS \
     --kind StorageV2 \
     --min-tls-version TLS1_2 \
     --allow-blob-public-access false
   ```

2. Create a container:
   ```bash
   az storage container create \
     --name evidence \
     --account-name myevidencestorage \
     --auth-mode login
   ```

3. Enable blob versioning (recommended):
   ```bash
   az storage account blob-service-properties update \
     --account-name myevidencestorage \
     --resource-group forensics-rg \
     --enable-versioning true
   ```

4. Get storage account key:
   ```bash
   az storage account keys list \
     --account-name myevidencestorage \
     --resource-group forensics-rg \
     --query "[0].value" -o tsv
   ```
   Or generate a SAS token with write permissions scoped to the container.

5. Minimum permissions if using Azure AD auth: `Storage Blob Data Contributor` role on the container.

**SPECTER config:**

```yaml
output:
  remote: az://evidence/cases/IR-2026-0042/
credentials:
  azure_storage_account: myevidencestorage
  azure_storage_key: ${AZURE_STORAGE_KEY}
upload:
  concurrent_uploads: 3
  verify_after_upload: true
imaging:
  segment_preset: cloud
```

If `azure_storage_key` is omitted, SPECTER attempts passwordless auth (managed identity, Azure CLI credentials).

## Upload Settings

The following settings control upload behavior across all providers. These are placed under the `upload` key in your YAML config.

| Setting | Default | Description |
|---------|---------|-------------|
| `max_bandwidth_mbps` | `50` | Maximum upload bandwidth in Mbps. Set to `0` for unlimited. |
| `concurrent_uploads` | `3` | Number of simultaneous file uploads. |
| `retry_max_attempts` | `10` | Maximum retry attempts per failed upload. |
| `retry_backoff_seconds` | `30` | Initial backoff interval in seconds. Uses exponential backoff with a maximum interval of 5 minutes. |
| `verify_after_upload` | `true` | Verify uploaded file integrity (HEAD request + checksum comparison) after each upload completes. |

## Configuration Reference

### YAML Config (`specter.yaml`)

Used by headless mode. Place in the working directory or specify with `--config`:

```yaml
examiner: "Examiner Name"
description: "Case description"
output_dir: "/path/to/output"
source_disk: "/dev/sdX"
segment_size: "4G"
compression: fast
phases:
  - memory
  - volatile
  - triage
  - disk
output:
  remote: s3://bucket/prefix/
credentials:
  aws_access_key_id: ${AWS_ACCESS_KEY_ID}
  aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  aws_region: us-east-1
upload:
  max_bandwidth_mbps: 50
  concurrent_uploads: 3
  retry_max_attempts: 10
  retry_backoff_seconds: 30
  verify_after_upload: true
imaging:
  segment_preset: default
```

Credential fields support `${ENV_VAR}` expansion for secure configuration. If the `credentials` section is omitted, each cloud SDK falls back to its default credential chain (AWS IAM roles, GCP Application Default Credentials, Azure managed identity).

### Segment Sizes

| Size | Preset | Use Case |
|------|--------|----------|
| 2G | `cloud` | Cloud uploads, legacy FAT32 media, older systems |
| 4G | `default` | Optimal for modern filesystems (NTFS, APFS, ext4) |
| 8G | `large` | Large disks, fast storage, fewer files |

### Imaging Options

| Option | Values | Description |
|--------|--------|-------------|
| `segment_preset` | `cloud`, `default`, `large` | Selects the segment size preset (see table above) |
| `segment_size` | `2G`, `4G`, `8G` | Explicit segment size (overrides preset) |
| `compression` | `none`, `fast`, `best` | E01 compression level |

## Requirements

- **Administrator/root** -- required for raw disk access and memory capture
- **All platforms** -- fully self-contained. All forensic tools (`ewfacquire`, `ewfverify`, WinPmem) are bundled in the `tools/` directory. No package installation is needed or permitted -- installing software on the target machine would taint the forensic image.
- Memory tools (optional): WinPmem (Windows, bundled), osxpmem (macOS/Intel, bundled), LiME (Linux, kernel module)

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `ewfacquire: command not found` | ewfacquire not in PATH or `tools/` | Verify the `tools/` directory exists next to the SPECTER binary and contains `ewfacquire`. Re-extract from the release zip if missing. |
| `permission denied` / `access denied` | Not running as admin/root | Run with `sudo` (macOS/Linux) or as Administrator (Windows) |
| Disk imaging progress stuck at 0% | Stale TUI state (older versions) | Update to latest SPECTER release |
| Acquisition hangs after all phases complete | Older version missing completion event | Update to latest SPECTER release |
| `seal: permission denied` on evidence files | Files locked as read-only after sealing | On macOS: `chflags -R nouchg /path/to/output` then re-seal. On Windows: run as Administrator. |
| Memory capture fails on macOS Apple Silicon | osxpmem not supported on ARM Macs | Memory capture is not available on Apple Silicon. Use an Intel Mac or skip the memory phase. |
| Upload fails with timeout | Network instability | SPECTER retries automatically (default 10 attempts with exponential backoff). Check network connectivity. Reduce `concurrent_uploads` if bandwidth is constrained. |
| `invalid S3 URI` / `invalid GCS URI` / `invalid Azure URI` | Malformed remote path | Use full URI format: `s3://bucket/prefix/`, `gs://bucket/prefix/`, `az://container/prefix/` |
| Resume starts disk phase over | Disk phase was interrupted mid-segment | This is expected. Disk imaging restarts from the beginning because partial E01 segments cannot be resumed. Completed phases are skipped. |
| E01 verification fails after imaging | Disk contents changed during acquisition | Re-acquire with the source disk write-blocked. Use a hardware write blocker for forensic soundness. |
| `azure_storage_account is required` | Missing Azure credential config | Set `azure_storage_account` in the credentials section or via `AZURE_STORAGE_ACCOUNT` env var |
| Large file hashing appears frozen | Hashing large files (physmem.raw, E01) takes time | Progress updates appear every 2 seconds for files over 100MB. Wait for completion. |

## Evidence Collection Guide

For forensic best practices, chain of custody procedures, legal considerations,
and cloud storage security guidance, see the
[Evidence Collection Guide](docs/EVIDENCE-GUIDE.md).

**Important:** Consult qualified legal counsel in your jurisdiction before using
SPECTER for evidence collection in legal proceedings.

## License

Proprietary. (c) 2026 Tech Javelin, Ltd. All Rights Reserved.

## Links

- **Website**: https://techjavelin.com
- **GitHub**: https://github.com/techjavelin/specter-app
- **Evidence Guide**: [docs/EVIDENCE-GUIDE.md](docs/EVIDENCE-GUIDE.md)
