# Dropbox — File Storage & Sync

Source: hellointerview.com problem-breakdowns/dropbox (free).

## Requirements

**Functional:** upload files from any device; download from any device; share files with other users; auto-sync across devices.
**Out of scope:** in-place file editing, preview-without-download.

**Non-functional:** high availability (favor over consistency); files up to 50GB; secure and recoverable; fast upload/download/sync.
**Out of scope:** storage quotas, versioning, virus scanning.

## Core entities

File (raw bytes), FileMetadata (name, size, MIME type, uploader), User.

## API

```
POST /files/presigned-url         → presigned S3 upload URL
GET  /files/{fileId}               → metadata
GET  /files/{fileId}/presigned-url → presigned S3 download URL
POST /files/{fileId}/share         → shares with another user
GET  /files/changes?since=         → sync delta
```

## High-level design (escalation per feature)

- **Upload:** single server (bad, doesn't scale/reliable) → through-backend to S3 (good, but backend is a redundant bottleneck) → **direct presigned-URL upload to S3**, with an S3 event notification updating metadata after the fact (great).
- **Download:** through-backend (bad) → direct presigned S3 download (good) → **presigned URL through a CDN** for geographic locality (great).
- **Sharing:** sharelist embedded in file metadata (bad — slow reverse "files shared with me" queries) → cached user→files mapping (good) → **normalized `SharedFiles(userId, fileId)` table** (great).
- **Sync:** local→remote is just the upload APIs triggered by a filesystem watcher; remote→local is a hybrid of WebSocket push (real-time) with periodic polling as a fallback for missed events.

## Deep dive 1 — large files (up to 50GB)

At 100Mbps, 50GB is over an hour — timeouts, gateway payload caps (e.g. API Gateway's 10MB), and network interruptions are all real constraints, not edge cases.
- **Chunking:** split client-side into 5-10MB pieces → parallel + resumable upload, progress UI.
- **Fingerprinting:** SHA-256 of the whole file (dedup) and of each chunk (resumability — know which chunks already landed).
- **Chunk status — Good vs. Great:** trusting a client-reported "this chunk is uploaded" PATCH is a security/correctness hole (good, but exploitable); the great version has the server verify against S3's own ETags via `ListParts` before marking a chunk done.
- **Mechanism:** S3 multipart upload API — create multipart upload → presigned URL per part → client uploads parts in parallel → client reports part ETags → backend verifies and calls `CompleteMultipartUpload`.

## Deep dive 2 — speeding uploads/syncs up further

- Client-side compression before encryption (skip already-compressed media); Content-Defined Chunking (rolling-hash chunk boundaries instead of fixed-size) so a small edit only invalidates the chunks actually near it — this is what makes efficient delta-sync possible, not naive fixed-size chunking.
- HTTP Range requests for parallel/resumed downloads (native S3 + CDN support).

## Deep dive 3 — security

HTTPS in transit; S3 server-side encryption at rest; access control via the `SharedFiles` table; short-TTL (~5 min) signed URLs so a leaked link expires quickly; CDN-level signed-URL support (CloudFront signing) for the cached-download path.

## Final architecture

Clients (with local sync agent) → Load Balancer/API Gateway → File Service (presigned-URL issuance, metadata orchestration) → DynamoDB (FileMetadata, SharedFiles) + S3 (blobs) → CloudFront (cached downloads).

## Level expectations

- **Mid (E4):** clean API/data model, functional upload/download/share design; responds reasonably when probed about resumability but may not know presigned URLs or chunking unprompted.
- **Senior (E5):** drives the large-file deep dive unprompted, knows the S3 multipart API specifically, explores tradeoffs proactively; ~60% breadth / 40% depth.
- **Staff+ (L6+):** ~40% breadth / 60% depth; anticipates problems from real experience; may redirect the conversation toward the area they can go deepest on.
