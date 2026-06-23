# SPECTER Changelog

All notable changes to SPECTER will be documented in this file.
This changelog is client-facing and included in public releases.

## v1.1.0 -- Split-Mode Acquisition, API Collection, Cold Boot

### New: Acquisition Modes

- **Live Capture** -- volatile-only mode (memory + processes/network) for IT staff
  to run before closing a device for transport. Saves state for later resume.
- **Cold Collection** -- examiner boots from WinPE USB and resumes where live
  capture left off. Completes triage, API, disk imaging, and sealing.
- **Auto-Resume** -- automatically detects incomplete acquisitions across all
  mounted volumes and continues without manual path selection.

### New: Hibernate File Extraction

- Detects and preserves `hiberfil.sys` during cold acquisition
- Validates full hibernate vs fast-startup (header magic inspection)
- Provides RAM-equivalent memory snapshot without requiring live access

### New: API Collection Phase

- Plugin-based provider architecture for external data integrations
- **SentinelOne** -- agent info, threat timeline, deep visibility events, activity log
- **Microsoft Intune** -- device compliance, installed apps, config profiles,
  BitLocker recovery keys, audit logs
- Configurable via TUI (inline-expanding config forms) or YAML config file
- Shared HTTP transport with retry, rate limit handling, and response logging

### New: Cold Boot Environment (WinPE)

- `specter coldboot prepare` generates bootable WinPE USB media
- Branded boot menu with SPECTER ASCII art and 3-option selection
- BitLocker unlock integration (recovery key in memory only, never persisted)
- Requires Windows ADK on examiner's prep workstation

### New: Context-Aware Wizard

- Operation menu adapts to boot environment (live vs cold)
- Disabled items shown with reason, cursor skips them
- Live-to-Cold transition: optional volatile capture before reboot

### Documentation

- Integration guides for SentinelOne and Intune
- MDM deployment guide (Intune, Jamf, JumpCloud)
- Cloud targets guide (S3, Azure Blob, GCS)
- Updated Technical Reference, Evidence Guide, and Working with Evidence

## v1.0.0 -- Initial Release

### Forensic Acquisition Engine

SPECTER is a portable, cross-platform forensic disk and memory acquisition toolkit
built for incident response professionals, digital forensic examiners, and
managed security teams.

### Features

- **Interactive TUI** -- guided wizard interface with real-time progress, live
  log output, and intuitive keyboard navigation
- **E01 Disk Imaging** -- forensically sound Expert Witness Format acquisition
  with hardware-level integrity verification via ewfacquire/ewfverify
- **Memory Capture** -- volatile memory acquisition using platform-native tools
  (WinPmem on Windows, osxpmem on macOS)
- **Cloud Upload** -- direct evidence upload to AWS S3, Google Cloud Storage, or
  Azure Blob Storage with configurable credentials and bucket/container targeting
- **Cryptographic Sealing** -- HMAC-SHA256 integrity seal across all collected
  artifacts with tamper-evident manifest generation
- **Resume Support** -- interrupted acquisitions can be resumed from the last
  completed phase without re-imaging
- **Headless Mode** -- fully automated operation via YAML configuration for
  scripted or MDM-driven deployments
- **JSON Manifest** -- machine-readable `specter-manifest.json` with full chain
  of custody metadata, file hashes, and schema-validated structure
- **Schema Validation** -- JSON Schema contract for manifest files ensures
  interoperability and programmatic verification

### Platform Support

- Windows (amd64) -- fully self-contained with bundled ewfacquire, ewfverify,
  and WinPmem
- macOS (arm64, amd64) -- fully self-contained with bundled ewfacquire and
  ewfverify built from source
- Linux (amd64) -- fully self-contained with statically linked ewfacquire and
  ewfverify

### Portability

All platforms ship as a single zip archive containing the SPECTER binary and all
required forensic tools. No package installation is needed or permitted --
installing software on the target machine would modify the drive being imaged
and compromise forensic integrity.

### Deployment

- Extract to USB drive or network share
- Run as administrator/root
- Supports JumpCloud, Microsoft Intune, and Jamf Pro MDM deployment
- Cloud provider setup guides included for AWS, GCS, and Azure
