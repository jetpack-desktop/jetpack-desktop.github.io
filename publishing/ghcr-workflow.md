---
url: https://jetpack-desktop.github.io/publishing/ghcr-workflow.md
description: >-
  Automated GitHub Actions workflow template for validating, packaging, pushing
  JetPack apps to ghcr.io via ORAS, and verifying manifest_sha256.
---

# GitHub Actions Publishing to GHCR

This guide shows how to automate publishing an individual JetPack worker application to **GitHub Container Registry (`ghcr.io`)** using `jetpack` / `jetpack-daemon` and `oras`, then verify the remote `manifest_sha256` digest required by `catalog.signed.json`.

A ready-to-copy workflow file is included in the repository at `examples/publish-app-ghcr.yml`.

## 1. Prerequisites

1. A repository containing a valid JetPack worker (`app.toml`, `worker.py` or native binary, and optional built `ui/` bundle).
2. GitHub Actions permissions with `packages: write` so ${{ secrets.GITHUB_TOKEN }} can push OCI manifests and blobs to `ghcr.io`.
3. **Public package visibility**: JetPack Store installs pull from `ghcr.io` using anonymous Bearer token exchange (`https://ghcr.io/token`). After the first push, open the package settings on GitHub (`https://github.com/users/<owner>/packages/container/<package>/settings` or your organization's Packages settings) and set visibility to **Public** so `jetpack-daemon` can pull it on end-user machines without credentials.

## 2. Workflow Walkthrough

1. **Validate `app.toml`**:
   ```bash
   jetpack validate ./app.toml
   ```
2. **Package an ORAS-compatible OCI Image Layout v1.0.0 bundle**:

   ```bash
   jetpack package . --out ./dist/oci --oci-ref ghcr.io/your-org/my-worker:0.1.0
   ```

   `jetpack package` writes `oci-layout`, `blobs/sha256/*`, and root `index.json` with `"org.opencontainers.image.ref.name": "0.1.0"` pointing to either:

   * the multi-arch `OciImageIndex` blob (`application/vnd.oci.image.index.v1+json`) when `[hardware].architectures` lists multiple platforms, or
   * the single-arch `OciImageManifest` blob (`application/vnd.oci.image.manifest.v1+json`) when one architecture is declared.

   It also prints the exact `manifest_sha256`, the `oras cp --from-oci-layout` push command, and the `oras manifest fetch --descriptor` verification command.
3. **Push to `ghcr.io` with `oras`**:
   ```bash
   echo "${{ secrets.GITHUB_TOKEN }}" | oras login ghcr.io -u "${{ github.actor }}" --password-stdin
   oras cp --from-oci-layout ./dist/oci:0.1.0 ghcr.io/your-org/my-worker:0.1.0
   ```
4. **Verify the live remote digest**:
   ```bash
   oras manifest fetch --descriptor ghcr.io/your-org/my-worker:0.1.0
   ```
   Confirm the returned `digest` equals `manifest_sha256` from `jetpack package` before submitting your `CatalogAppEntry` to [Store Submission & Catalog Signing](/publishing/store-submission).
5. **Run the live end-to-end pull smoke test** (once the GHCR package is **Public**):
   ```bash
   jetpack-daemon \
     --oci-remote-smoke-test ghcr.io/your-org/my-worker:0.1.0 \
     --expected-digest sha256:<64-hex>
   ```
   This verifies anonymous `ghcr.io/token` Bearer negotiation, multi-arch platform selection, `307` CDN blob streaming, `SHA-256` CAS commit, and Tar-Slip-safe unpacking in an isolated temporary directory.

## 3. Complete Workflow Template (`examples/publish-app-ghcr.yml`)

```yaml
name: Publish JetPack App to GHCR

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  package-and-publish-ghcr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oras-project/setup-oras@v1
        with:
          version: "1.2.2"
      - uses: dtolnay/rust-toolchain@stable

      - name: Install JetPack CLI & Daemon
        run: |
          cargo install --locked --git https://github.com/jetpack-desktop/jetpack.git jetpack-cli jetpack-daemon

      - name: Validate & Package OCI Layout
        id: pkg
        run: |
          TAG="${GITHUB_REF_NAME#v}"
          OCI_REF="ghcr.io/$(echo "${GITHUB_REPOSITORY}" | tr '[:upper:]' '[:lower:]'):${TAG}"
          jetpack validate ./app.toml
          OUT="$(jetpack package . --out ./dist/oci --oci-ref "${OCI_REF}")"
          echo "${OUT}"
          echo "tag=${TAG}" >> "${GITHUB_OUTPUT}"
          echo "oci_ref=${OCI_REF}" >> "${GITHUB_OUTPUT}"
          echo "digest=$(echo "${OUT}" | awk '/manifest_sha256:/ {print $2; exit}')" >> "${GITHUB_OUTPUT}"

      - name: Push to GHCR & Verify Digest
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | oras login ghcr.io -u "${{ github.actor }}" --password-stdin
          oras cp --from-oci-layout "./dist/oci:${{ steps.pkg.outputs.tag }}" "${{ steps.pkg.outputs.oci_ref }}"
          REMOTE_DIGEST="$(oras manifest fetch --descriptor "${{ steps.pkg.outputs.oci_ref }}" | jq -r '.digest')"
          test "${REMOTE_DIGEST}" = "${{ steps.pkg.outputs.digest }}"
```

See also [CLI Tools Reference](/publishing/cli-tools) and [Store Submission & Catalog Signing](/publishing/store-submission).
