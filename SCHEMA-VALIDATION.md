# SPECTER Manifest Schema -- Validation Guide

This document explains how to locate and use the correct JSON Schema to validate a `specter-manifest.json` file produced by the SPECTER forensic acquisition toolkit.

## Overview

Every sealed SPECTER evidence package contains a `specter-manifest.json` at its root. This manifest is the canonical chain-of-custody record. Before importing an evidence package, the platform **must** validate the manifest against the published JSON Schema to ensure structural integrity.

## Schema Location

Schemas are published in the public repository: **`techjavelin/specter-docs`**

Each SPECTER release tags this repository with the same version tag (e.g., `v1.0.0`, `v1.0.0-RC`). The schema file is always at:

```
schemas/specter-manifest.schema.json
```

## How to Resolve the Schema URL

### Step 1: Read `specter_version` from the manifest

```json
{
  "specter_version": "1.0.0-RC",
  "run_id": "SutaJavelin-1782015113397301600",
  ...
}
```

The `specter_version` field matches the release version (the Git tag without the `v` prefix).

### Step 2: Derive the tag

The tag is simply `v` + `specter_version`. For example:
- `specter_version: "1.0.0-RC"` -> tag `v1.0.0-RC`
- `specter_version: "1.0.0"` -> tag `v1.0.0`
- `specter_version: "1.1.0"` -> tag `v1.1.0`

There is no separate version mapping. The version IS the version.

### Step 3: Fetch the schema

**Via raw URL (public, no auth required):**

```
https://raw.githubusercontent.com/techjavelin/specter-docs/v{specter_version}/schemas/specter-manifest.schema.json
```

Example:
```
https://raw.githubusercontent.com/techjavelin/specter-docs/v1.0.0-RC/schemas/specter-manifest.schema.json
```

**Via GitHub API:**

```http
GET https://api.github.com/repos/techjavelin/specter-docs/contents/schemas/specter-manifest.schema.json?ref=v{specter_version}
Accept: application/vnd.github.raw+json
```

No authentication is required -- this is a public repository.

**Fallback:** If the exact tag is not found (e.g., a dev build), fetch from the `latest` release or `main` branch.

### Step 4: Validate

Use any JSON Schema 2020-12 compatible validator:

- **Python:** `jsonschema` library
- **JavaScript/TypeScript:** `ajv` with `ajv-formats`
- **Go:** `santhosh-tekuri/jsonschema`
- **Ruby:** `json_schemer`

Example (Python):
```python
import json
import jsonschema
import urllib.request

def validate_manifest(manifest_path, schema_tag=None):
    with open(manifest_path) as f:
        manifest = json.load(f)

    # Derive tag from manifest version if not specified
    if schema_tag is None:
        schema_tag = "v" + manifest["specter_version"]

    schema_url = f"https://raw.githubusercontent.com/techjavelin/specter-docs/{schema_tag}/schemas/specter-manifest.schema.json"

    with urllib.request.urlopen(schema_url) as r:
        schema = json.loads(r.read())

    jsonschema.validate(manifest, schema)  # raises ValidationError on failure
```

Example (TypeScript):
```typescript
import Ajv from "ajv";
import addFormats from "ajv-formats";

async function validateManifest(manifest: Record<string, unknown>) {
  const version = manifest.specter_version as string;
  const schemaTag = `v${version}`;
  const schemaUrl = `https://raw.githubusercontent.com/techjavelin/specter-docs/${schemaTag}/schemas/specter-manifest.schema.json`;

  const res = await fetch(schemaUrl);
  const schema = await res.json();

  const ajv = new Ajv({ strict: false });
  addFormats(ajv);
  const validate = ajv.compile(schema);

  if (!validate(manifest)) {
    throw new Error(`Manifest validation failed: ${JSON.stringify(validate.errors)}`);
  }
}
```

## Key Validation Rules

The schema enforces:

- **All required fields** must be present (`specter_version`, `run_id`, `examiner`, `host`, `phases`, `files`, `seal`, etc.)
- **`sealed` must be `true`** -- unsealed packages must not be imported
- **Every file entry** must have either `sha256` (hex, 64 chars) **or** `verified_by` (string) -- never both, never neither
- **Phase statuses** must be one of: `pending`, `running`, `done`, `failed`, `skipped`
- **No additional properties** are allowed at any level -- the schema is strict
- **HMAC seal** must include `hmac_sha256` (64-char hex) and `hmac_key_components` array

## Post-Validation: HMAC Verification

Schema validation confirms structure. To verify **integrity**, the importing application should also:

1. Re-derive the HMAC key from `seal.hmac_key_components` (join values with `|`)
2. Serialize the seal document **without** `hmac_sha256` and `hmac_key_components`
3. Compute HMAC-SHA256 and compare to `seal.hmac_sha256`

This confirms the manifest has not been tampered with since sealing.
