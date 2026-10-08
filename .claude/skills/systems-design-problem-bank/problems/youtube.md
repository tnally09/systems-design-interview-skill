# YouTube — Video Upload & Streaming

Source: hellointerview.com problem-breakdowns/youtube (free). Distinct from `top-k-videos.md`, which is a narrower analytics problem in the same setting.

## Requirements

**Functional:** upload a video; watch (stream) a video.
**Out of scope:** view counts, search, comments, recommendations, channels/subscriptions.

**Non-functional:** availability over consistency; support very large uploads (10s of GB); low-latency streaming even on constrained bandwidth; ~1M uploads/day, ~100M views/day; resumable uploads.

## Core entities

User, Video, VideoMetadata (uploader, transcript URLs, etc).

## API

```
POST /presigned_url   { VideoMetadata } → presigned S3 URL     (upload)
GET  /videos/{videoId} → VideoMetadata + segment URLs           (streaming)
```

## High-level design (escalation)

**Upload:** store the raw file as-is (bad — incompatible with the range of playback devices/bandwidths) → store multiple pre-transcoded formats (good — but only supports whole-file downloads, not partial streaming) → **store each format as short segments** (great — enables both adaptive quality and partial/streaming playback, at the cost of a real processing pipeline).

**Streaming:** full-file download (bad — 10GB at 100Mbps is 13+ minutes, and any network blip loses all progress) → incremental segment downloads (good — but ignores that network conditions change mid-playback) → **adaptive bitrate streaming** (great — client fetches a manifest listing available quality variants, and switches between them live based on measured network conditions).

## Deep dive 1 — the post-processing pipeline (this is the real content of the problem)

Modeled as a DAG, not a linear pipeline: split the source into segments (ffmpeg) → transcode each segment into every target format in parallel → generate manifest files (a primary manifest plus one per format/variant) → mark the upload complete.
- An orchestrator (e.g. Temporal) coordinates the DAG's dependencies and lets independent segment-transcode jobs run in parallel across worker nodes.
- Transcoding is CPU-intensive — this is the actual scaling bottleneck of the whole pipeline, not the upload itself.
- Intermediate artifacts live in S3 between stages (workers never talk to each other directly), and S3 event notifications trigger the next DAG stage.

## Deep dive 2 — resumable uploads

Client splits the file into 5-10MB chunks with per-chunk fingerprints; `VideoMetadata` tracks each chunk's upload status; chunks upload via S3's multipart upload API (in parallel); the client reports back each chunk's S3-issued ETag/part number for backend verification (mirrors Dropbox's "verify via S3, don't trust the client" pattern — see `dropbox.md`). On an interrupted upload, the client re-fetches `VideoMetadata`, sees which chunks are already confirmed, and resumes only the missing ones.

## Deep dive 3 — scaling to 1M uploads/day, 100M views/day

Per-component reasoning is the expected answer shape here, not a single silver bullet:
- Video (metadata) Service: stateless, horizontally scaled behind a load balancer.
- Metadata store (Cassandra): partitioned by `videoId`; a popular video's metadata reads can still create a hot partition — mitigate with replication across nodes and a distributed LRU cache layer in front, partitioned the same way.
- Video processing: queue-driven, elastic worker pool that scales on queue depth.
- S3: elastic by default for a single region; cross-region replication needed only if the product genuinely needs geo-distributed origin storage.
- Playback latency: a CDN caches both segments and manifest files at the edge, so a popular video's steady-state playback never touches the origin/backend at all once cached.

## Final architecture

Direct-to-S3 presigned upload → DAG-orchestrated post-processing (segment + multi-format transcode + manifest) → Cassandra metadata (partitioned, cached) → CDN-served adaptive-bitrate playback.

## Level expectations

- **Mid (80% breadth / 20% depth):** clean API/data model, functional upload+playback design, aware of multipart upload and segment-based streaming at a surface level, can drive clarity on at least one deep dive.
- **Senior (60% breadth / 40% depth):** moves quickly through the high-level design specifically to spend real time on the post-processing pipeline, explains multipart-upload resumability concretely, articulates the tradeoffs made.
- **Staff+ (40% breadth / 60% depth):** breezes fundamentals, goes deep on the DAG orchestration and scaling strategy specifically, draws on real experience, treats the interviewer as a peer.
