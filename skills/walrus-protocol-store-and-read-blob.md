---
name: Store and read a blob on Walrus
description: Upload bytes to Walrus via a publisher, capture the returned blob ID, then read the bytes back from an aggregator.
api: openapi/walrus-protocol-publisher-openapi.json, openapi/walrus-protocol-aggregator-openapi.json
operations: [put_blob, get_blob, get_blob_byte_range]
---

# Store and read a blob on Walrus

Walrus is content-addressed, immutable blob storage. You **write** through a
*publisher* daemon and **read** through an *aggregator* daemon. Both are plain HTTP;
no authentication is required against the public endpoints.

## Endpoints
- Publisher (write): `https://publisher.walrus-mainnet.walrus.space` (or `-testnet-` for the sandbox)
- Aggregator (read): `https://aggregator.walrus-mainnet.walrus.space` (or `-testnet-`)

## Steps

1. **Store the bytes** — `put_blob`: `PUT /v1/blobs` with the raw body as
   `application/octet-stream`. Set `?epochs=N` for how long to keep it (default 1).
   ```bash
   curl -X PUT "$PUBLISHER/v1/blobs?epochs=5" --data-binary @myfile.bin
   ```
2. **Read the result** — the JSON `BlobStoreResult` is either `newlyCreated`
   (contains the new Sui `blob_object` and its `blobId`) or `alreadyCertified`
   (the blob already existed — see idempotency below). Extract the `blobId`.
3. **Read the bytes back** — `get_blob`: `GET /v1/blobs/{blob_id}` returns the raw
   bytes. For a partial fetch use `get_blob_byte_range`
   (`GET /v1/blobs/{blob_id}/byte-range`).
   ```bash
   curl "$AGGREGATOR/v1/blobs/$BLOB_ID" --output out.bin
   ```

## Rules the agent must follow
- **Idempotency is by content.** Re-storing identical bytes returns `alreadyCertified`
  rather than duplicating — retries are safe. Pass `force=true` only to force a new object.
- **Everything is public.** Any blob is world-readable and discoverable. Encrypt with
  Seal before storing anything sensitive.
- **Handle transient reads.** A `503` on `get_blob` is transient (mid-upload/expiry or
  nodes temporarily unreachable) — retry with backoff. `404` means not-yet-stored or
  unknown blob ID; `451` means the blob is blocked.
- **Error envelope** is a JSON `Status` object `{error:{status,code,message,details}}`
  (not RFC 9457). See `errors/walrus-protocol-problem-types.yml`.
- **Public publisher limits.** Public publishers are best-effort and size-limited
  (`413` too large, `504` store timeout); run your own publisher for production writes.
