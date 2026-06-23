# SPECTER Technical Reference

## Tools, Collection Methods, and Admissibility Justification

**Document Version:** 1.0.0
**Last Updated:** 2026-06-21

---

## IMPORTANT LEGAL NOTICE

This document describes the technical implementation of SPECTER's evidence
collection process. It is provided for informational purposes only and does not
constitute legal advice. Consult qualified legal counsel in your jurisdiction
regarding the admissibility of digital evidence.

Tech Javelin, Ltd. accepts no responsibility or liability for any legal
consequences arising from the use of this software. See the
[LICENSE](LICENSE) for complete terms.

---

## 1. Disk Imaging

### Tool: ewfacquire / ewfverify (libewf)

| Attribute | Detail |
|-----------|--------|
| Tool | ewfacquire (acquisition), ewfverify (verification) |
| Project | libewf (https://github.com/libyal/libewf) |
| License | GNU Lesser General Public License v3.0 (LGPL-3.0) |
| Output Format | Expert Witness Format (E01 / EnCase 6) |
| Bundled | Yes -- all platforms |

### Why ewfacquire

The Expert Witness Format (E01) is an industry-standard forensic disk image
format originally developed for EnCase, one of the most widely used commercial
forensic platforms. The E01 format is accepted by courts, law enforcement
agencies, and forensic laboratories worldwide.

The libewf implementation is an open-source library that reads and writes E01
files. It is used by multiple forensic platforms and has been in continuous
development since 2006. Its open-source nature means the acquisition process
is transparent and auditable -- there is no proprietary black box between the
source disk and the output image.

**Key properties of the E01 format:**

- **Bit-for-bit imaging**: Creates an exact sector-level copy of the source media.
- **Internal checksums**: CRC32 checksums are computed for each data chunk during
  acquisition, enabling detection of corruption at any point in the image.
- **Integrated hash verification**: MD5 and/or SHA-1 hashes of the complete
  source data are computed during acquisition and stored in the image header.
- **Compression**: Supports configurable compression to reduce storage
  requirements without affecting image integrity.
- **Segmentation**: Large images can be split into segments of configurable size
  for storage on media with file size limitations.
- **Metadata embedding**: Case number, examiner name, evidence description, and
  acquisition notes are stored within the image file.

### How SPECTER Uses ewfacquire

SPECTER invokes ewfacquire with the following parameters:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `-f encase6` | EnCase 6 format | Widely recognized E01 format |
| `-C` | "SPECTER acquisition - {hostname}" | Case description embedded in image |
| `-D` | "{disk model} {serial}" | Device description |
| `-e` | Examiner name (user-provided) | Examiner identification |
| `-E` | "SPECTER {version} run_id={id}" | Evidence note with tool version and run ID |
| `-S` | Segment size in MiB (default: 4096) | Segment file size limit |
| `-c` | Compression method (default: "fast") | Balance between speed and size |
| `-u` | Unattended mode | Non-interactive operation |

After imaging, SPECTER runs `ewfverify` on the resulting E01 file to
independently confirm that the image data matches the source checksums computed
during acquisition.

### Admissibility Support

- E01 format is accepted by EnCase, FTK, Autopsy, X-Ways, and other major
  forensic platforms for analysis.
- The format includes internal integrity verification mechanisms (CRC32 per
  chunk, MD5/SHA-1 of complete image).
- SPECTER additionally computes SHA-256 hashes of the E01 segment files and
  records them in the chain-of-custody manifest.
- The open-source nature of libewf allows independent verification of the
  acquisition process.

---

## 2. Memory Capture

### Windows: WinPmem

| Attribute | Detail |
|-----------|--------|
| Tool | WinPmem (winpmem_mini_x64.exe) |
| Project | https://github.com/Velocidex/WinPmem |
| License | Apache License 2.0 |
| Output Format | Raw physical memory dump (physmem.raw) |
| Bundled | Yes -- Windows only |

WinPmem is an open-source memory acquisition tool developed by the Velocidex
team (creators of Velociraptor). It uses a signed kernel driver to read physical
memory, producing a raw memory dump that can be analyzed with Volatility,
Rekall, or other memory forensics frameworks.

**Why WinPmem:**
- Open-source with auditable source code.
- Uses a signed kernel driver, allowing deployment without disabling driver
  signature enforcement.
- Produces standard raw memory format compatible with all major analysis tools.
- Maintained by the Velocidex project, which is actively used in enterprise
  incident response.

### macOS: osxpmem

| Attribute | Detail |
|-----------|--------|
| Tool | osxpmem |
| Project | Part of the pmem suite |
| License | Apache License 2.0 |
| Output Format | Raw physical memory dump |
| Bundled | Yes -- macOS Intel only |

osxpmem uses a kernel extension (kext) to access physical memory on macOS. It
is limited to Intel-based Macs. Apple Silicon (ARM64) Macs do not support
third-party kexts, and Apple does not provide a mechanism for physical memory
acquisition on these platforms.

**Limitation:** Memory capture is not available on Apple Silicon Macs. SPECTER
will skip the memory phase on ARM64 macOS systems and log a notification.

### Linux: /proc/kcore

| Attribute | Detail |
|-----------|--------|
| Tool | dd (reading /proc/kcore) |
| Source | Linux kernel pseudofilesystem |
| License | N/A (kernel facility) |
| Output Format | Raw memory dump |
| Bundled | N/A (uses system dd) |

On Linux, SPECTER reads `/proc/kcore`, a pseudofile exposed by the kernel that
provides access to physical memory. This approach requires root privileges but
does not require loading a kernel module.

**Alternative:** LiME (Linux Memory Extractor) provides a dedicated kernel
module for memory acquisition. LiME must be compiled against the running kernel
and is not bundled with SPECTER. Organizations requiring Linux memory acquisition
in production should pre-build LiME modules for their kernel versions.

---

## 3. Volatile State Collection

Volatile state data is collected using a combination of the
[gopsutil](https://github.com/shirou/gopsutil) library (cross-platform) and
platform-specific system commands.

### Collected Artifacts

| Artifact | Method | Format | Justification |
|----------|--------|--------|---------------|
| Running processes | gopsutil Process enumeration | JSON | Captures PID, name, executable path, command line, username, parent PID, creation time, and status. Documents what was executing at time of collection. |
| Network connections | gopsutil net.Connections | JSON | Records all active TCP/UDP connections with local/remote addresses, ports, state, and associated PID. Establishes network activity at time of collection. |
| Network interfaces | gopsutil net.Interfaces | JSON | Records interface configuration (IP addresses, MAC addresses, flags). Documents network identity of the device. |
| DNS cache | ipconfig /displaydns (Windows), dscacheutil (macOS), system resolver (Linux) | Text | Records recently resolved domain names. May indicate communication with command-and-control or exfiltration endpoints. |
| ARP table | arp -a (all platforms) | Text | Records IP-to-MAC address mappings. Documents devices on the local network segment that the target has communicated with. |

### Why Volatile Collection Matters

Volatile data exists only in the running state of the operating system. Once a
device is powered off, this data is lost. Per NIST SP 800-86 Section 4.2, the
order of volatility should guide collection priority: data that will be lost
soonest should be collected first.

SPECTER collects volatile state before disk imaging to preserve this ephemeral
evidence. The collected data is written to the output directory and included in
the sealed manifest with SHA-256 hashes.

### Defensibility

- All volatile collection uses read-only operations. No data is written to the
  target disk.
- The gopsutil library reads from kernel-provided interfaces (/proc on Linux,
  syscalls on Windows/macOS) and does not modify system state.
- Command-line tools (ipconfig, arp) are standard operating system utilities
  that perform read-only queries.
- Collection timestamps are recorded in the manifest to establish when the data
  was captured.

---

## 4. Triage Collection

Triage data consists of system artifacts that provide investigative context.
These are files and data stores that exist on disk but are critical for rapid
analysis.

### Windows Artifacts

| Artifact | Source | Method | Justification |
|----------|--------|--------|---------------|
| Registry hives | HKLM\SYSTEM, HKLM\SOFTWARE, HKLM\SAM, HKLM\SECURITY, user NTUSER.DAT | File copy (reg save for system hives) | Registry hives contain system configuration, user activity, installed software, network history, USB device history, and other forensically significant data. |
| BitLocker keys | Volume recovery information | PowerShell (manage-bde) | Recovery keys are needed to access encrypted volumes during analysis. Without them, disk image contents may be inaccessible. |
| Event logs | System, Security, Application, PowerShell, Sysmon, Task Scheduler, Terminal Services, Windows Defender, WinRM | File copy (.evtx) | Event logs record authentication events, process execution, network activity, and security events. Critical for timeline reconstruction. |
| Browser artifacts | Chrome/Edge: History, Downloads, Bookmarks, Login Data, Cookies | File copy (SQLite databases) | Browser artifacts document web activity, downloads, and potentially compromised credentials. |
| Installed software | Registry Uninstall keys (32-bit and 64-bit) | Registry query | Documents what software was installed, including potentially unauthorized or malicious tools. |
| Prefetch files | C:\Windows\Prefetch\*.pf | File copy | Prefetch files record application execution history with timestamps, providing evidence of program execution even after the program has been removed. |

### macOS Artifacts

| Artifact | Source | Method | Justification |
|----------|--------|--------|---------------|
| System logs | Unified log (last 24 hours) | log show command | Unified log captures system events, application activity, and security-relevant events. |
| Browser artifacts | Chrome, Safari | File copy | Documents web activity and downloads. |
| Installed software | System Profiler | system_profiler SPApplicationsDataType | Records all installed applications with versions and paths. |

### Linux Artifacts

| Artifact | Source | Method | Justification |
|----------|--------|--------|---------------|
| Journal logs | systemd journal (last 7 days) | journalctl | System journal records service activity, authentication events, and system errors. |
| Browser artifacts | Chrome, Firefox | File copy | Documents web activity and downloads. |
| Installed software | Package manager | dpkg -l or rpm -qa | Records all installed packages. |

### Defensibility

- Triage collection reads existing files and queries system APIs. No data is
  written to the target disk.
- File copies preserve original metadata (timestamps, permissions) where the
  filesystem supports it.
- All collected artifacts are hashed (SHA-256) and recorded in the manifest.
- The specific artifacts collected are documented in this reference to support
  the examiner's testimony about what was collected and why.

---

## 5. Integrity and Sealing

### SHA-256 Hashing

Every artifact collected by SPECTER is hashed using SHA-256 after collection.
SHA-256 is a cryptographic hash function published by NIST (FIPS 180-4). It is
widely accepted in forensic practice and legal proceedings as a means of
demonstrating data integrity.

Disk images (E01 files) use a `verified_by` field that records the result of
ewfverify rather than re-hashing potentially multi-terabyte files. ewfacquire
and ewfverify perform their own internal hash verification during the imaging
process, and SPECTER records this verification result in the manifest.

### HMAC-SHA256 Seal

After all phases complete, SPECTER computes an HMAC-SHA256 seal across the
complete manifest. The HMAC key is deterministically derived from case metadata:

```
key = run_id | examiner | hostname | serial | sealed_at
```

(Fields joined with `|` delimiter)

This seal provides:
- **Tamper detection**: Any modification to the manifest or its recorded hashes
  invalidates the seal.
- **Deterministic verification**: The seal can be independently recomputed from
  the same case metadata to verify integrity.
- **No external key management**: The key is derived from the evidence metadata
  itself, eliminating the need for key storage or distribution.

### Manifest Structure

The sealed manifest (`specter-manifest.json`) records:
- SPECTER version used for acquisition
- Examiner name
- Device hostname and serial number
- Run ID (unique per acquisition)
- Start time, end time, and seal time
- SHA-256 hash for each collected artifact
- Verification status for each disk image
- HMAC-SHA256 seal value

The manifest is validated against a published JSON Schema
(`specter-manifest.schema.json`) before writing.

---

## 6. Tool Licensing Summary

| Tool | License | Implication |
|------|---------|-------------|
| ewfacquire / ewfverify (libewf) | LGPL-3.0 | Permissive for bundling. LGPL allows linking and distribution with proprietary software provided the LGPL-licensed library can be replaced by the user. SPECTER bundles pre-compiled binaries. |
| WinPmem | Apache-2.0 | Permissive. Allows use, modification, and distribution with attribution. |
| osxpmem | Apache-2.0 | Permissive. Same terms as WinPmem. |
| gopsutil | BSD-3-Clause | Permissive. Allows use and redistribution with attribution. |

All bundled tools use open-source licenses that permit redistribution in
commercial products. License compliance requires attribution, which is provided
in this document and in the SPECTER binary's help output.

SPECTER itself is proprietary software. See the [LICENSE](LICENSE) for terms.

---

## References

- libewf project: https://github.com/libyal/libewf
- Expert Witness Format specification: documented in libewf source and EnCase documentation
- WinPmem project: https://github.com/Velocidex/WinPmem
- gopsutil project: https://github.com/shirou/gopsutil
- NIST FIPS 180-4 (SHA-256): https://csrc.nist.gov/publications/detail/fips/180/4/final
- NIST SP 800-86: https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-86.pdf

---

## 7. Acquisition Modes

SPECTER supports three acquisition modes that accommodate the split workflow
between IT staff (on-site) and forensic examiners (off-site):

### Full Acquisition

Standard mode -- all phases run in sequence on a single session:
`MEMORY -> VOLATILE -> TRIAGE -> API (optional) -> DISK -> SEAL`

### Live Capture (Volatile Only)

Designed for IT staff to run before closing a device for transport:
- Captures only MEMORY and VOLATILE phases
- Saves state file for later resume
- No disk imaging, no seal
- CLI: `specter acquire --headless --live --config specter.yaml`

### Cold Collection (Resume from Live Capture)

Examiner boots the device from WinPE USB media and resumes:
`HIBERNATE -> TRIAGE -> API (optional) -> DISK -> SEAL`

The hibernate phase extracts `hiberfil.sys` before anything touches the disk,
preserving the RAM state from when the device was hibernated/closed.

### Phase Ordering

| Phase | Mode: Full | Mode: Live | Mode: Cold |
|-------|-----------|-----------|-----------|
| HIBERNATE | -- | -- | First (if available) |
| MEMORY | 1st | 1st | Skipped (done in live) |
| VOLATILE | 2nd | 2nd | Skipped (done in live) |
| TRIAGE | 3rd | -- | 2nd |
| API | 4th (optional) | -- | 3rd (optional) |
| DISK | 5th | -- | 4th |
| SEAL | 6th | -- | 5th |

---

## 8. Hibernate File Extraction

### What is hiberfil.sys

When Windows hibernates (lid close, explicit hibernate, or hybrid shutdown),
the kernel writes the contents of physical RAM to `C:\hiberfil.sys`. This file
is a compressed snapshot of all memory at the time of hibernation.

### Forensic Value

A full hibernate file contains the same data as a live memory dump:
- Complete process tree and execution state
- Open network connections and sockets
- Encryption keys (BitLocker FVEK, cached credentials, session tokens)
- Loaded DLLs, kernel drivers, rootkit indicators
- Clipboard, open file handles, registry in memory

### Detection and Validation

SPECTER validates the hibernate file by checking:
1. **Header magic bytes** (first 4 bytes):
   - `hibr` or `HIBR` = Full hibernate (complete RAM snapshot)
   - `wake` = Fast Startup / hybrid shutdown (kernel-only, limited value)
2. **File size** relative to installed RAM (full hibernate is typically 40-100% of RAM)

### Handling

- If full hibernate detected: copied with progress reporting to `memory/hiberfil.sys`
- If fast startup detected: copied with warning about limited forensic value
- If not found: phase skipped with log entry (non-fatal)
- Metadata written to `memory/hiberfil-metadata.txt`

---

## 9. API Collection Phase

### Architecture

The API collection phase uses a plugin-based provider architecture:

```
Provider Interface -> Registry -> Collector -> Phase Engine
```

Each provider implements:
- `ID()` / `Name()` / `Description()` -- identity
- `ConfigFields()` -- defines the configuration form (drives both TUI and YAML)
- `Validate(cfg)` -- pre-flight configuration check
- `Collect(ctx, cfg, outputDir, events)` -- executes collection

### Built-in Providers

| Provider | ID | Data Collected |
|----------|-----|---------------|
| SentinelOne | `sentinelone` | Agent info, threats, deep visibility events, activity log |
| Microsoft Intune | `intune` | Compliance, installed apps, config profiles, BitLocker keys, audit logs |

### Configuration (YAML)

```yaml
api_integration:
  sentinelone:
    api_url: "https://usea1-partners.sentinelone.net"
    api_token: "your-api-token"
    hostname: "WORKSTATION-01"
    options:
      - agent_info
      - threats
      - deep_visibility
      - activity_log
  intune:
    tenant_id: "your-tenant-id"
    client_id: "app-client-id"
    client_secret: "app-secret"
    device_name: "WORKSTATION-01"
    options:
      - compliance
      - installed_apps
      - recovery_keys
```

### Output Structure

```
evidence/api/
  sentinelone/
    agent_info.json
    threats.json
    deep_visibility.json
    activity_log.json
    _transport_logs/
  intune/
    device_info.json
    compliance.json
    installed_apps.json
    recovery_keys.json
    audit_logs.json
    _transport_logs/
```

### Adding New Providers

See the [API Provider Development Guide](../agents.md) for implementation details.
Individual provider documentation: [SentinelOne](integrations/SENTINELONE.md),
[Intune](integrations/INTUNE.md).

---

## 10. Cold Boot Environment (WinPE)

### Overview

SPECTER can generate a bootable WinPE USB drive for cold acquisition. The boot
environment includes the SPECTER binary, forensic tools, and a branded boot menu.

### Boot Menu

On boot, the WinPE environment displays the SPECTER ASCII banner and presents:

```
1. Auto-Resume (detect and continue from live capture)
2. New Acquisition (full wizard)
3. Boot with Config File (specter.yaml from boot media)
```

- Option 1 scans all accessible volumes for `specter-state.json`
- Option 2 launches the full interactive TUI wizard
- Option 3 reads config from boot media root; falls back to wizard if missing

### Building Boot Media

Requires Windows ADK installed on the examiner's preparation workstation.

```
specter coldboot prepare --output F:\boot.iso
specter coldboot prepare --output F:\boot.iso --config specter.yaml --tools ./tools
```

### BitLocker Unlock

When the target volume is BitLocker-encrypted, SPECTER prompts for the recovery
key during cold acquisition. The key is:
- Held in memory only during the session
- Never written to state files, logs, or evidence output
- Passed directly to `manage-bde -unlock` on WinPE

### Boot Environment Detection

SPECTER detects its execution context automatically:
- **Live (Native OS):** Normal execution on a running system
- **Cold USB (WinPE):** Detected via WinPE registry markers or removable media heuristics
- **Cold Network (PXE):** Reserved for future network boot scenarios

The wizard menu adapts accordingly -- Live Capture is disabled in cold boot,
Cold Collection is disabled when not running from removable media.

---

## References (continued)

- Windows Hibernation format: Microsoft documentation, Volatility hibernation layer
- SentinelOne API: https://usea1-partners.sentinelone.net/apidoc/
- Microsoft Graph API: https://learn.microsoft.com/en-us/graph/api/overview
- Windows ADK: https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install
- manage-bde: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde

---

*Copyright 2026 Tech Javelin, Ltd. All rights reserved.*
*This document is provided for informational purposes only and does not
constitute legal advice.*
