# MDM Deployment Guide

## Remote Deployment of SPECTER for Live Collection

**Document Version:** 1.0.0
**Last Updated:** 2026-06-23

---

## Overview

Mobile device management and endpoint management platforms can be used to push
SPECTER to target systems for rapid live capture when physical access is not
available. This is particularly useful for distributed workforces, remote
incident response, and time-sensitive preservation efforts where waiting for a
hands-on examiner would risk evidence loss.

An MDM-driven deployment should be treated as an operational response mechanism,
not as a substitute for legal authorization, scope review, and chain-of-custody
documentation.

---

## General Deployment Model

The common MDM workflow is:

1. Package the SPECTER binary and any approved supporting files.
2. Deliver a configuration file containing output destination settings and
   examiner-identifying metadata.
3. Trigger a silent acquisition command on the endpoint.
4. Direct output to a pre-configured evidence destination such as a network share
   or cloud object store.
5. Verify that the evidence package arrived intact.

For remote deployments, use headless execution and a deterministic output
destination so the examiner can validate collection without interactive user
prompts.

---

## Microsoft Intune Deployment

### Intune for Windows

For Windows endpoints, package SPECTER as a **Win32 app**.

#### Packaging Flow

1. Obtain the SPECTER Windows release ZIP.
2. Extract the package contents into a staging directory.
3. Place `specter.exe`, supporting binaries, and the approved configuration file
   in the package source directory.
4. Use **IntuneWinAppUtil.exe** to wrap the directory into an `.intunewin`
   package.
5. Upload the resulting package in the Intune admin center as a Win32 app.

#### Detection Rule

Use a simple file-exists detection rule, for example:

- Path: `C:\Program Files\SPECTER\`
- File: `specter.exe`

This confirms the binary is present at the expected install path.

#### Requirement Rules

Set requirement rules to limit deployment to supported Windows versions. For
example:

- Windows 10 or later
- 64-bit architecture
- Minimum build level consistent with your SPECTER release requirements

#### Assignments

Target the app to the appropriate **device group** rather than broad tenant-wide
assignment. Use tightly scoped response groups for incident-driven deployments.

#### Configuration File Delivery

You can deliver the configuration file in one of two common ways:

- **Embedded in the package** so it installs together with the binary
- **Deployed separately** through proactive remediation, script delivery, or a
  separate file-placement workflow

Store the configuration in a stable location such as:

`C:\ProgramData\specter\specter.yaml`

#### Silent Execution

Use a non-interactive command such as:

```powershell
specter.exe acquire --headless --live --config C:\ProgramData\specter\specter.yaml
```

Configure the output destination ahead of time so evidence is written directly to
a network share or S3-compatible target without user intervention.

### Intune for macOS

For macOS endpoints, deployment is typically script-driven.

Common patterns include:

- Distributing the binary with a shell script
- Wrapping the binary into a `.pkg` using `pkgbuild`
- Delivering settings via a custom configuration profile, including a
  `.mobileconfig` where appropriate for managed settings

A typical workflow is:

1. Build or obtain the macOS SPECTER package.
2. Wrap it into a `.pkg` if your organization standardizes on package deployment.
3. Use an Intune shell script or package assignment to install SPECTER.
4. Deliver the configuration through a managed profile or file deployment step.
5. Trigger headless execution through script delivery or a follow-up action.

---

## Jamf Pro Deployment

Jamf Pro is well-suited for macOS response workflows.

Recommended approach:

- Package SPECTER as a **payload-free package** when you want the install process
  to be script-controlled
- Use a **Policy** to deploy the package
- Add a **script payload** to place the configuration file and run the command
- Optionally expose the action in **Self Service** if the collection depends on
  user coordination or support-assisted initiation

Self Service should only be used when that fits the investigative plan. For
time-sensitive response, automated policy-triggered execution is generally more
predictable.

---

## JumpCloud Deployment

JumpCloud can be used to deploy SPECTER through command execution and file
distribution workflows.

### Windows

Use **PowerShell** commands to:

- Create the install directory
- Copy the SPECTER binary and configuration file
- Execute the acquisition command

### macOS and Linux

Use **Bash** commands to:

- Create the target directory
- Place the binary and configuration file
- Set executable permissions
- Launch the acquisition

JumpCloud is often best suited for smaller or mixed-platform environments where
script-based control is sufficient.

---

## Credential Handling

**Never deploy recovery keys via MDM.**

The SPECTER configuration file delivered through MDM should contain only what is
needed for the acquisition workflow, such as:

- Output destination
- Examiner name or case metadata
- Operational collection options

If a recovery key is required for a cold-boot or post-restart workflow, it
should be provided separately by the examiner through a controlled process and
must not be pre-positioned in a broadly deployed MDM package.

---

## Post-Collection Verification

After remote execution completes, verify collection by checking the configured
output location.

Recommended steps:

1. Confirm the evidence directory or object prefix was created.
2. Verify expected core artifacts are present, such as:
   - `specter-manifest.json`
   - `specter-state.json`
   - Disk, memory, or triage output directories as applicable
3. Run SPECTER verification against the collected package where practical.
4. Record the acquisition completion time, destination path, and integrity
   results in the case record.

### Remote Integrity Verification

If the evidence is collected to a network share or cloud destination, perform a
remote integrity check by comparing the manifest-listed SHA-256 values with the
uploaded files. This helps confirm the package was not corrupted in transit.

---

## Bandwidth Considerations

The SPECTER binary is approximately **15 MB**, so deployment overhead is minimal
for most enterprise networks.

Network impact is driven more by the resulting evidence output than by the binary
itself. When pushing remote live collection at scale:

- Stagger large deployments
- Use pre-approved evidence destinations near the target region when possible
- Consider bandwidth-throttling options for remote uploads

---

## Related Documentation

- [Cloud Targets Guide](./CLOUD-TARGETS.md)
- [Evidence Collection Guide](../EVIDENCE-GUIDE.md)
- [Working with SPECTER Evidence Artifacts](../WORKING-WITH-EVIDENCE.md)
- [SPECTER Technical Reference](../TECHNICAL-REFERENCE.md)
