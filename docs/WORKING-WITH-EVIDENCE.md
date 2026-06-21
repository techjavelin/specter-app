# Working with SPECTER Evidence Artifacts

## Guidance for Third-Party Examiners and Legal Teams

**Document Version:** 1.0.0
**Last Updated:** 2026-06-21

---

## IMPORTANT

This document provides general guidance for handling evidence collected by
SPECTER. It does not constitute legal advice. Consult qualified legal counsel
regarding chain of custody requirements in your jurisdiction.

**Do not modify original evidence files.** Always work from verified copies.
Any modification to the original artifacts will invalidate the HMAC integrity
seal and may compromise admissibility.

---

## 1. Evidence Package Structure

A SPECTER evidence package is a directory containing:

```
specter-output/
├── specter-manifest.json      # Chain-of-custody manifest (sealed)
├── specter-state.json         # Acquisition state (operational, not evidentiary)
├── disk/
│   ├── evidence.E01           # Forensic disk image (E01 format)
│   ├── evidence.E02           # Additional segments (if image exceeds segment size)
│   └── ...
├── memory/
│   └── physmem.raw            # Physical memory dump (raw format)
├── volatile/
│   ├── processes.json         # Running processes at time of capture
│   ├── network_connections.json
│   ├── network_interfaces.json
│   ├── dns_cache.txt
│   └── arp_table.txt
└── triage/
    ├── registry/              # (Windows) SAM, SYSTEM, SOFTWARE, SECURITY, NTUSER.DAT
    ├── event_logs/            # (Windows) .evtx files
    ├── browser_artifacts/     # Browser history, downloads, cookies (SQLite)
    ├── prefetch/              # (Windows) .pf files
    ├── installed_software.txt
    └── system_logs/           # (macOS/Linux) unified log or journal
```

Not all directories will be present in every package. The phases executed during
acquisition are recorded in `specter-manifest.json`.

---

## 2. Verifying Package Integrity

Before working with any artifacts, verify the integrity seal.

### 2.1 Using SPECTER (Recommended)

```bash
specter verify /path/to/specter-output
```

This re-computes SHA-256 hashes of all files listed in the manifest and verifies
the HMAC-SHA256 seal. A successful verification confirms that no artifacts have
been modified since the package was sealed.

### 2.2 Manual Hash Verification

If SPECTER is not available, you can verify individual file hashes manually:

```bash
# Read expected hashes from manifest
cat specter-manifest.json | jq '.files[] | "\(.sha256)  \(.path)"'

# Compute actual hash of a specific file
sha256sum volatile/processes.json
# or on macOS:
shasum -a 256 volatile/processes.json
```

Compare the computed hash against the value in the manifest. Any mismatch
indicates the file has been modified.

### 2.3 Build Provenance Verification

Release binaries are attested using GitHub artifact attestation (Sigstore). To
verify that a SPECTER binary is an authentic, unmodified build:

```bash
gh attestation verify specter_1.0.0_darwin_arm64.zip --repo techjavelin/specter-app
```

---

## 3. Disk Images (E01)

### 3.1 Format

SPECTER produces Expert Witness Format (E01 / EnCase 6) disk images using
`ewfacquire`. E01 is an industry-standard format supported by all major forensic
platforms.

E01 images include:
- Bit-for-bit copy of the source disk
- Internal CRC32 checksums per data chunk
- MD5/SHA-1 hash of the complete source data
- Embedded case metadata (examiner, description, notes)
- Optional compression and segmentation

### 3.2 Verifying E01 Integrity

Before mounting or analyzing, verify the image's internal checksums:

```bash
# Using ewfverify (bundled with SPECTER)
ewfverify evidence.E01

# Using Autopsy/Sleuth Kit
img_stat evidence.E01
```

A successful ewfverify confirms the image data matches the checksums computed
during acquisition. This is independent of SPECTER's manifest -- it is a
property of the E01 format itself.

### 3.3 Mounting for Read-Only Access

**Always mount forensic images read-only.** Writing to a mounted image
constitutes tampering.

**Linux (using ewfmount):**
```bash
# Mount E01 as a raw device file
mkdir /mnt/ewf
ewfmount evidence.E01 /mnt/ewf

# The raw image appears at /mnt/ewf/ewf1
# Mount the filesystem within it (read-only)
mkdir /mnt/disk
mount -o ro,loop,noexec /mnt/ewf/ewf1 /mnt/disk
```

**Linux (using xmount for format conversion):**
```bash
# Convert to raw or VHD for tool compatibility
xmount --in ewf evidence.E01 --out raw /mnt/converted
```

**Using forensic platforms:**
- **Autopsy** (open source): File > Add Data Source > Disk Image
- **FTK Imager** (free): File > Add Evidence Item > Image File
- **EnCase**: Add Evidence > Add Evidence File
- **X-Ways Forensics**: File > Open > Image File

These tools handle E01 natively and enforce read-only access.

### 3.4 Working with Copies

If you need to perform analysis that might modify data (e.g., running malware
scans, recovering deleted files):

1. Verify the original E01 hash against the manifest
2. Create a working copy of the image
3. Perform analysis on the copy only
4. Document that analysis was performed on a copy, not the original
5. Retain the original unmodified for court presentation

---

## 4. Memory Dumps (physmem.raw)

### 4.1 Format

Memory dumps are raw physical memory captures. On Windows, these are produced by
WinPmem. The file is a byte-for-byte copy of physical RAM at the time of capture.

### 4.2 Analysis Tools

| Tool | License | Platform | Notes |
|------|---------|----------|-------|
| Volatility 3 | BSD-like | Cross-platform | Industry standard, plugin-based |
| Rekall | GPLv2 | Cross-platform | Alternative framework |
| MemProcFS | AGPL-3.0 | Windows/Linux | Memory as a mounted filesystem |
| Strings | Open source | Cross-platform | Basic string extraction |

### 4.3 Common Analysis with Volatility 3

```bash
# List running processes
vol -f physmem.raw windows.pslist

# Show network connections
vol -f physmem.raw windows.netscan

# Extract command line arguments
vol -f physmem.raw windows.cmdline

# Detect injected code
vol -f physmem.raw windows.malfind

# List loaded DLLs
vol -f physmem.raw windows.dlllist

# Extract files from memory
vol -f physmem.raw windows.filescan
vol -f physmem.raw windows.dumpfiles --virtaddr <address>
```

For Linux memory dumps:
```bash
vol -f physmem.raw linux.pslist
vol -f physmem.raw linux.bash
vol -f physmem.raw linux.netscan
```

### 4.4 Integrity Considerations

- Memory dumps are point-in-time captures. The system was running during
  acquisition, so some memory contents may have changed between the start
  and end of the capture.
- Verify the SHA-256 hash in the manifest before analysis.
- Memory analysis tools do not modify the dump file (read-only operations).
- Note the capture timestamp from the manifest when correlating memory
  artifacts with other evidence.

---

## 5. Volatile State (JSON)

### 5.1 Processes (processes.json)

Contains an array of process objects with:
- `pid` -- Process ID
- `name` -- Process name
- `exe` -- Full executable path
- `cmdline` -- Command line arguments
- `username` -- User account running the process
- `ppid` -- Parent process ID
- `create_time` -- Process start time
- `status` -- Running/sleeping/etc.

**Use cases:** Identify suspicious processes, establish what was executing at
collection time, correlate with memory analysis findings.

```bash
# Find processes by name
cat processes.json | jq '.[] | select(.name | test("powershell|cmd|python"; "i"))'

# Find processes running as SYSTEM
cat processes.json | jq '.[] | select(.username == "NT AUTHORITY\\SYSTEM")'
```

### 5.2 Network Connections (network_connections.json)

Contains active TCP/UDP connections with local/remote addresses, ports, state,
and associated PID.

```bash
# Find established connections to external IPs
cat network_connections.json | jq '.[] | select(.status == "ESTABLISHED")'

# Find connections by PID
cat network_connections.json | jq '.[] | select(.pid == 1234)'
```

### 5.3 DNS Cache and ARP Table

Plain text files captured from system utilities. Use standard text search tools:

```bash
# Search DNS cache for suspicious domains
grep -i "suspicious-domain" dns_cache.txt

# Review ARP entries
cat arp_table.txt
```

---

## 6. Triage Artifacts

### 6.1 Windows Registry Hives

Registry hives are binary files. Use dedicated tools for analysis:

| Tool | Purpose |
|------|---------|
| RegRipper | Automated registry analysis with plugins |
| Registry Explorer (Eric Zimmerman) | GUI-based registry browsing |
| python-registry | Programmatic access (Python) |
| regipy | Registry parsing library (Python) |

Common investigative queries:
- **USB device history:** SYSTEM\CurrentControlSet\Enum\USBSTOR
- **Recently run programs:** SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\UserAssist
- **Network profiles:** SOFTWARE\Microsoft\Windows NT\CurrentVersion\NetworkList
- **Installed services:** SYSTEM\CurrentControlSet\Services
- **Autorun entries:** SOFTWARE\Microsoft\Windows\CurrentVersion\Run

### 6.2 Windows Event Logs (.evtx)

Event logs are in Windows XML Event Log format. They cannot be read with
standard text tools.

| Tool | Purpose |
|------|---------|
| Event Log Explorer | GUI viewer for .evtx |
| EvtxECmd (Eric Zimmerman) | Command-line parser, CSV/JSON output |
| python-evtx | Programmatic access (Python) |
| Chainsaw | Rapid threat hunting across evtx files |
| Hayabusa | Rule-based evtx analysis (Sigma rules) |

```bash
# Using Chainsaw for rapid triage
chainsaw hunt event_logs/ --sigma-rules /path/to/sigma/rules

# Using Hayabusa
hayabusa csv-timeline -d event_logs/ -o timeline.csv

# Using python-evtx for targeted queries
python -c "
import Evtx.Evtx as evtx
with evtx.Evtx('event_logs/Security.evtx') as log:
    for record in log.records():
        if '4624' in record.xml():  # Logon events
            print(record.xml())
"
```

Key event IDs:
- **4624/4625** -- Successful/failed logon
- **4688** -- Process creation (if auditing enabled)
- **7045** -- Service installed
- **1102** -- Audit log cleared
- **4720** -- User account created

### 6.3 Browser Artifacts

Browser history, downloads, and cookies are SQLite databases. Use any SQLite
client:

```bash
# Chrome/Edge history
sqlite3 History "SELECT url, title, last_visit_time FROM urls ORDER BY last_visit_time DESC LIMIT 50;"

# Chrome/Edge downloads
sqlite3 History "SELECT target_path, tab_url, start_time FROM downloads ORDER BY start_time DESC;"
```

Timestamps in Chrome databases are WebKit format (microseconds since
1601-01-01). Convert with:
```
unix_timestamp = (webkit_timestamp / 1000000) - 11644473600
```

### 6.4 Prefetch Files (.pf)

Prefetch files record application execution history on Windows. They are binary
files requiring specialized tools:

| Tool | Purpose |
|------|---------|
| PECmd (Eric Zimmerman) | Command-line prefetch parser |
| WinPrefetchView (NirSoft) | GUI viewer |
| prefetch (Python library) | Programmatic access |

```bash
# Parse all prefetch files to CSV
PECmd.exe -d prefetch/ --csv output/ --csvf prefetch_results.csv
```

Prefetch files record:
- Executable name and path
- Run count
- Last 8 execution timestamps
- Files/directories accessed by the application

---

## 7. Maintaining Chain of Custody

When receiving a SPECTER evidence package from another party:

### 7.1 Receiving Evidence

1. **Document the transfer** -- record who provided the evidence, when, by what
   method (physical media, secure file transfer, cloud download).
2. **Verify integrity immediately** -- run `specter verify` or manually check
   hashes before doing anything else.
3. **Log the verification result** -- record that integrity was confirmed at
   time of receipt.
4. **Store securely** -- place in access-controlled storage with logging.

### 7.2 During Analysis

1. **Work on copies, not originals** -- create a working copy for analysis.
   Document the copy creation including source hash verification.
2. **Document all actions** -- maintain a log of what analysis was performed,
   by whom, when, and what tools were used.
3. **Do not modify originals** -- never write to, rename, or reorganize files
   in the original evidence directory.

### 7.3 Transferring to Third Parties

1. **Re-verify integrity** before transfer -- confirm hashes still match.
2. **Document the transfer** -- chain of custody log entry with recipient
   name, date, time, purpose, and method.
3. **Use secure transfer methods** -- encrypted channels, tamper-evident
   physical packaging, or WORM cloud storage with access logging.
4. **Provide this document** -- so the recipient knows how to work with the
   artifacts without compromising integrity.

### 7.4 What Invalidates Integrity

The following actions will break the HMAC seal and/or file hashes:

- Modifying, renaming, or deleting any file in the evidence directory
- Extracting files from the E01 image into the evidence directory
- Adding analysis notes or reports into the evidence directory
- Opening and saving files (some editors modify metadata on save)
- Mounting the E01 image in read-write mode

**Store analysis outputs in a separate directory, never alongside the
original evidence.**

---

## 8. Providing Evidence to Legal Counsel or Courts

When evidence must be presented in legal proceedings:

1. **Provide the complete, unmodified evidence package** -- all files as
   collected, with the manifest.
2. **Provide verification instructions** -- a copy of this document or the
   `specter verify` command.
3. **Provide the manifest** -- `specter-manifest.json` contains the complete
   record of what was collected, when, by whom, and with what tools.
4. **Provide tool documentation** -- the Technical Reference
   (`docs/TECHNICAL-REFERENCE.md`) documents the tools and methods used.
5. **Be prepared to testify** about the collection process, including:
   - Your qualifications and authorization
   - The device state at time of collection
   - The tools used (SPECTER version, as recorded in manifest)
   - How integrity was maintained from collection to presentation

---

## References

- Volatility 3: https://github.com/volatilityfoundation/volatility3
- Eric Zimmerman's Tools: https://ericzimmerman.github.io/
- Chainsaw: https://github.com/WithSecureLabs/chainsaw
- Hayabusa: https://github.com/Yamato-Security/hayabusa
- Autopsy: https://www.autopsy.com/
- libewf/ewftools: https://github.com/libyal/libewf
- SPECTER Technical Reference: [docs/TECHNICAL-REFERENCE.md](TECHNICAL-REFERENCE.md)
- SPECTER Evidence Guide: [docs/EVIDENCE-GUIDE.md](EVIDENCE-GUIDE.md)

---

*Copyright 2026 Tech Javelin, Ltd. All rights reserved.*
*This document is provided for informational purposes only and does not
constitute legal advice.*
