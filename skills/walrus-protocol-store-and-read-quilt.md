---
name: Store and read a Walrus quilt (batched small files)
description: Batch many small files into a single Walrus quilt via a publisher, then read individual patches back by identifier from an aggregator.
api: openapi/walrus-protocol-publisher-openapi.json, openapi/walrus-protocol-aggregator-openapi.json
operations: [put_quilt, get_patch_by_quilt_id_and_identifier, get_patch_by_quilt_patch_id, list_patches_in_quilt]
---

# Store and read a Walrus quilt

A **quilt** batches many small files (each a *patch*) into one Walrus blob, cutting
per-blob overhead. Write with the publisher, read individual patches with the aggregator.

## Steps

1. **Store the quilt** — `put_quilt`: `PUT /v1/quilts` as `multipart/form-data`. Each
   form field is a file whose field name is its identifier; an optional `_metadata`
   field carries a JSON array of per-file `{identifier, tags}`.
   ```bash
   curl -X PUT "$PUBLISHER/v1/quilts?epochs=5" \
     -F "contract-v2=@document.pdf" \
     -F "logo-2025=@image.png"
   ```
   The `QuiltStoreResult` returns the quilt's `blobStoreResult` plus `storedQuiltBlobs[]`,
   each a `StoredQuiltPatch` with its `identifier` and `quiltPatchId`.
2. **List patches** (optional) — `list_patches_in_quilt`:
   `GET /v1/quilts/{quilt_id}/patches`.
3. **Read one patch** — either by human identifier with
   `get_patch_by_quilt_id_and_identifier`
   (`GET /v1/blobs/by-quilt-id/{quilt_id}/{identifier}`), or by patch ID with
   `get_patch_by_quilt_patch_id` (`GET /v1/blobs/by-quilt-patch-id/{quilt_patch_id}`).
   Patch bytes come back in the body; identifier/tags come back as `X-Quilt-*` and
   `ETag` headers.

## Rules the agent must follow
- Use quilts for **many small (<10 MB) files**; store large files as standalone blobs.
- Quilt reads and writes share the same public/idempotency/error semantics as blobs —
  see `conventions/walrus-protocol-conventions.yml` and
  `errors/walrus-protocol-problem-types.yml`.
- All patches are public; encrypt sensitive data before storing.
