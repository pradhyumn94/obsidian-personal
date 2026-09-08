Large files (>10MB) belong in object storage (S3/GCS/Azure Blob), not the DB, and not proxied through app servers. Rule of thumb: over 10MB and doesn't need SQL queries → blob storage.

## Why not DB or proxy-through-server
- BLOB columns kill query perf, backups, replication — object stores give ~11 nines durability, unlimited capacity, per-object pricing
- Naive path (client → API → blob storage, and reverse) makes app servers dumb pipes — pure latency/cost with zero value added, and the bottleneck

## Direct access pattern
Server's role shifts from data transfer to **access control**: validate the request, hand out temporary scoped credentials, get out of the way (ticket booth, not usher).

- **Presigned URL (upload)** — signed in-memory using the server's cloud credentials, no network call needed. Scoped to one file, one location, 15min–1hr expiry. Bake `content-length-range` and `content-type` into the signature so it can't be abused. Client does a plain `PUT` — server never sees the bytes
- **Signed URL (download)** — direct from blob storage (cheap, simple, infrequent access) or via CDN (cached at edge, better for hot files). Blob-storage signatures are checked by the storage service with your secret key; CDN signatures use public/private key crypto validated at the edge with no origin callback

## Resumable / multipart uploads
Large files dropping mid-transfer shouldn't restart from zero (5GB @ 100Mbps ≈ 7min — losing that at 99% is expensive).
- Split into parts (each gets a presigned URL or a range-addressed chunk), client tracks the session ID, queries which parts succeeded, resumes from the first missing one
- Progress = parts done / total parts, for free
- Completion call lists part numbers + checksums; storage assembles into one object — until then, parts exist but no file does. Set lifecycle rules to clean up abandoned multipart sessions (24-48hrs) since they still cost money

## State synchronization
DB and blob storage are two systems updating independently — trusting the client to report "done" causes races (DB says completed before file exists), orphaned files (client crashes before notifying), fake completions (malicious client), and lost notifications (network failure).

**Fix:** event notifications (storage service fires an event on actual object arrival, carrying the same `storage_key` you wrote to the DB row at presign time) as the primary update path, plus **reconciliation** (periodic job re-checks rows stuck in `pending` against real storage state) as the safety net for dropped events.

Metadata pattern: create the DB row as `status: pending` before the file exists, key like `uploads/{user_id}/{timestamp}/{uuid}` generated server-side (never client-supplied — collision/overwrite risk). Keep rich metadata in the DB, not object tags (S3 caps at 10×256 chars, unqueryable).

## Provider terminology
| Feature | AWS | GCP | Azure |
|---|---|---|---|
| Temporary upload URLs | Presigned URLs (PUT/POST) | Signed URLs (resumable/simple) | SAS tokens |
| Multipart uploads | Multipart Upload API (5MB–5GB parts) | Resumable Uploads (flexible chunks) | Block Blobs (4MB–100MB blocks) |
| Event notifications | S3 Event Notifications → Lambda/SQS/SNS | Cloud Storage Pub/Sub → Cloud Functions | Event Grid |
| CDN + signed URLs | CloudFront | Cloud CDN | Azure CDN |
| Cleanup policy | Lifecycle Rules | Lifecycle Management | Lifecycle Management Policies |

## When to use / skip
- **Use** for anything >10MB going through the API: video/photo uploads, file sync (Dropbox), chat media (WhatsApp)
- **Skip** for: small files (<10MB, normal endpoints), synchronous validation (e.g. CSV header check before accept — must see bytes), compliance/inspection (HIPAA, card-number scanning — must proxy, in chunks), instant-feedback UX (face detection on upload)

## Deep dives

### Upload fails at 99%?
Chunked/multipart uploads solve this at the provider level. Client queries which parts succeeded (`ListParts` in S3, resumable session status in GCS, committed blocks in Azure) and resumes from the first gap instead of restarting. Store the session ID client-side (e.g. `localStorage`) so resume survives app restarts.

### Preventing abuse?
Quarantine bucket first — virus/content scan before the file is reachable — then move to the public bucket and flip DB status to `available`. Always enforce `content-length-range` in the presigned URL conditions. The scan delay itself throttles abuse since an attacker gets no immediate signal that an upload "worked."

### Handling metadata?
DB row created at presign time (`pending`, server-generated key) is the source of truth; storage events update it to `completed` on arrival. Don't lean on object tags for anything you need to query.

### Fast downloads?
CDN solves geography (edge caching, ms latency after first pull). Range requests (`Range: bytes=0-N`) solve large-file resumability — client tracks completed ranges, re-requests only what's missing. Parallel chunk downloads (4-6 at once) can 3-4x throughput but rarely worth the complexity; most users are bandwidth not connection limited.

## Related
- [[Scaling Reads]] — CDN/edge caching is the same idea applied to bytes instead of query results
- [[Scaling Writes]] — async offloading via storage events mirrors write-offloading through a queue
- [[Redis]] — could track upload session/progress state, though providers handle this natively via the multipart session ID
- [[CAP Theorem]] — DB-vs-blob-storage sync is an eventual consistency problem: events are primary, reconciliation is the safety net
