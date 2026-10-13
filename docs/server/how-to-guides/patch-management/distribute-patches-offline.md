---
myst:
  html_meta:
    description: "How to distribute patches to an air-gapped or offline on-premises Livepatch Server using the patch uploader workflow."
---

(server-how-to-guides-distribute-patches-offline)=

# Distribute patches across an air gap

Livepatch on-premises deployments can operate in fully air-gapped environments without inbound or outbound internet access. The `livepatch-admin` command-line tool provides a complete offline patch distribution workflow:

1. **Download**: Query and download patches from Canonical's hosted Livepatch Server to a local directory on an internet-connected system.
2. **Transfer**: Move the patch directory across the air gap using approved physical media or secure network boundaries.
3. **Upload**: Upload patches and register their metadata with the air-gapped on-premises Livepatch Server.

## Prerequisites

- **On the internet-connected system:**
  - An installation of the Livepatch administration tool with the `livepatch-admin` alias configured as described in the [administration tool setup guide](/server/how-to-guides/security/setup-administration-tool.md).
  - An Ubuntu Pro token entitled to Livepatch, attached to the system or available as a machine token.
  - Outbound HTTPS network access to `https://livepatch.canonical.com`.

- **On the air-gapped network:**
  - Network connectivity to the on-premises Livepatch Server.
  - An authenticated administrator session created with `livepatch-admin login`.

## Step 1: Query and download patches from the hosted server

On the internet-connected workstation, list the patches available from Canonical's hosted server.

### List available patches

Use `livepatch-admin patch list` to inspect available patches before downloading:

```bash
livepatch-admin patch list --tier stable
```

You can narrow the results using filters:

- `--kernel <version>`: Filter by kernel release prefix (for example, `5.15.0` or `5.15.0-107.117-generic`).
- `--arch <architecture>`: Target architecture (`amd64`, `arm64`, `s390x`, `ppc64le`).
- `--flavor <flavor>`: Kernel flavour (for example, `generic`, `lowlatency`, `aws`, `azure`, `gcp`).
- `--published-after <date>`: RFC 3339 timestamp or `YYYY-MM-DD` date to view patches published after a specific point in time.
- `--published-before <date>`: RFC 3339 timestamp or `YYYY-MM-DD` date to view patches published before a specific point in time.
- `--tier <tier>`: Livepatch tier (defaults to `stable`).
- `--format <format>`: Output format (`table`, `json`, `yaml`).

### Download patches to disk

Once you identify the required patches, download them to a dedicated directory:

```bash
livepatch-admin patch download --patch-dir /tmp/livepatch-export --tier stable
```

Optional download flags include:

- `--patch-dir <path>`: Destination directory for downloaded patches (required).
- `--metadata-file <filename>`: Name of the metadata manifest file written inside `--patch-dir` (defaults to `canonical-livepatch-patch-metadata.json`).
- `--metadata-only`: Download only the metadata manifest file without downloading patch files.
- `--kernel <version>`: Download patches for kernels matching or newer than the specified version prefix (for example, `5.15.0`).
- `--arch <architecture>`: Restrict downloads to a specific CPU architecture.
- `--flavor <flavor>`: Restrict downloads to a specific kernel flavour.
- `--concurrency <int>`: Number of simultaneous patch downloads (defaults to `2`).
- `--retry-attempts <int>`: Number of retry attempts with exponential backoff if a download encounters a network interruption (defaults to `3`).

### Inspect the downloaded directory

The destination directory contains two types of artifacts:

```text
/tmp/livepatch-export/
├── livepatch-5.15.0-107.117-generic-107.1-amd64.tar.bz2
├── livepatch-5.15.0-107.117-generic-106.1-amd64.tar.bz2
└── canonical-livepatch-patch-metadata.json
```

- **Patch tarballs (`livepatch-*.tar.bz2`)**: Binary payload packages containing kernel livepatch modules.
- **Metadata manifest (`canonical-livepatch-patch-metadata.json`)**: A JSON manifest recording the kernel version, architecture, patch version, SHA-256 checksum, supported kernel list, and original Canonical publish timestamp for every downloaded patch.

```{important}
Do not alter or delete `canonical-livepatch-patch-metadata.json`. The upload command automatically reads this manifest to preserve original publish timestamps and supported kernel associations in your on-premises database.
```

## Step 2: Transfer the directory across the air gap

Copy the entire export directory—including all `.tar.bz2` archive files and `canonical-livepatch-patch-metadata.json`—to removable media (such as an encrypted USB drive or write-once optical media) or transfer it through your organisation's secure data diode or staging gateway.

Mount or copy the directory onto a workstation or bastion system with network connectivity to your on-premises Livepatch Server.

## Step 3: Upload patches to the on-premises server

Log in to your on-premises Livepatch Server with administrator credentials:

```bash
 export LIVEPATCH_URL=https://livepatch.internal.example.com
 livepatch-admin login --auth <username>:<password>
```

### Run the upload command

Navigate to the location of the transferred directory and upload the patches:

```bash
livepatch-admin patch upload --patch-dir /media/transfer/livepatch-export
```

### Command options and flags

- `--patch-dir <path>`: Local path to the directory containing patch archives (required).
- `--metadata-file <path>`: Path to the metadata manifest. Defaults to `canonical-livepatch-patch-metadata.json` inside `--patch-dir`.
  - When the default manifest is present, `livepatch-admin` automatically reads it to register patches with their original upstream publish timestamps and supported kernel lists.
  - If the default manifest is absent, `livepatch-admin` scans `--patch-dir`, parses filenames, and generates metadata dynamically, assigning the current system time as the publish timestamp.
  - If you pass an explicit `--metadata-file` path and the file is missing, the command immediately terminates with a fatal error.
- `--tier <tier>`: Destination tier for the uploaded patches. Defaults to the on-premises server's configured sync tier or the first configured tier (for example, `edge` or `stable`).
- `--concurrency <int>`: Number of parallel file uploads (defaults to `2`).
- `--overwrite`: Unconditionally overwrite existing patches on the server even if filenames or checksums already exist.
- `--cleanup`: Automated failure rollback. If an upload fails or server-side metadata registration encounters an error, any files uploaded during the session are automatically deleted from storage to avoid leaving the server in a partial state.

### Duplicate detection and checksum validation

Before uploading, `livepatch-admin` inspects the server's existing patch catalog to prevent redundant transfers or accidental collisions:

- **Matching checksum**: If the on-premises server already has a patch with an identical filename and matching SHA-256 checksum, the file is automatically skipped.
- **Checksum mismatch**: If a patch file has the same filename as an existing server patch but a differing SHA-256 checksum, the uploader halts processing for that file without modifying storage. It records the mismatch in `<patch-dir>/canonical-livepatch-patch-duplicates.json` for administrator review.
- **Force overwrite**: If you intentionally need to replace a mismatched patch, pass the `--overwrite` flag.

## Storage backend behavior

`livepatch-admin patch upload` works transparently across all supported storage backends:

- **Cloud object storage (AWS S3, Google Cloud Storage, Azure Blob Storage, IBM Cloud Object Storage, Oracle Cloud Infrastructure, OpenStack Swift):** The Livepatch Server issues temporary, scoped upload credentials to the administrator client. The client transfers binary payloads directly to cloud object storage using presigned POST policies, SAS tokens, or Pre-Authenticated Requests. Background workers automatically refresh credentials during large batch uploads before expiry.
- **Server-mediated storage (Local Filesystem, PostgreSQL):** The administrator client streams binary payloads directly through the Livepatch Server API (`PUT /v1/admin/patches/file/{filename}`) authenticated by the administrator's session token.

For details on configuring cloud permissions so your server can generate upload credentials, see the [Patch storage reference](/server/reference/patch-storage/index.md).
