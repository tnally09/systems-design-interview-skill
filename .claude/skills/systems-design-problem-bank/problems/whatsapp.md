# WhatsApp — Messaging

Source: hellointerview.com problem-breakdowns/whatsapp (free).

## Requirements

**Functional:** group chats (2-100 participants); send/receive messages; offline message storage (30 days); media attachments.
**Out of scope:** audio/video calls, business-account features, registration/onboarding.

**Non-functional:** delivery latency < 500ms; guaranteed deliverability; billions of users at high throughput; minimal server-side retention beyond the 30-day window; resilience to component failure.

## Core entities

User, Chat, Message, Client (a user can have multiple devices, each a separate Client).

## API

WebSocket-based, not REST — chosen because the traffic is genuinely bidirectional and high-frequency (see `systems-design-knowledge-base/reference/core-concepts.md`'s networking section for when that justification actually holds).
```
client→server: createChat, sendMessage, createAttachment, modifyChatParticipants
server→client: chatUpdate, newMessage
```
Plus an ACK mechanism so the server knows a message was actually received.

## High-level design

Chat creation: an L4 load balancer routes to a Chat Server, which writes Chat + ChatParticipant rows (DynamoDB, with a GSI for "which chats is this user in"). Sending a message: write to Messages + a per-recipient Inbox, attempt real-time delivery over the open WebSocket, client ACKs. Media: presigned URLs direct to S3, so attachments never transit the Chat Server.

## Deep dive 1 — scaling to billions of users

- **Bad:** naive horizontal scaling of Chat Servers breaks routing — nothing tells a sender's server which server holds the recipient's open connection.
- **Good:** consistent hashing assigns each user to a specific Chat Server, tracked via ZooKeeper/etcd.
- **Great:** offload routing to a sharded Redis Pub/Sub layer (sharded by user ID) so message delivery doesn't require Chat Servers to know about each other directly — a server publishes to the recipient's channel, whichever server holds that connection is subscribed and delivers it.

## Deep dive 2 — multiple devices per user

A `Clients` table per user, Inbox tracked per-client (not per-user), and a new message publishes to all of that user's connected clients, not just one.

## Deep dive 3 — WebSocket connection failures

- **Bad:** rely on the underlying TCP timeout to notice a dead connection — far too slow.
- **Good:** ACK timeouts trigger a server-side retry.
- **Great:** application-level heartbeats every 10-30s with a 5s timeout, so a dead connection is detected within ~15s regardless of what TCP itself thinks.

## Deep dive 4 — Redis Pub/Sub can silently drop messages

Pub/Sub has no persistence/replay — a subscriber that's briefly disconnected loses whatever was published in that window.
- **Good options:** periodic (30-60s) full-state polling as a backstop, or per-chat sequence numbers with gap detection.
- **Great:** piggyback the sequence number on the existing heartbeat, so a missed message is detected within one heartbeat interval rather than waiting for the next poll cycle.

## Deep dive 5 — message ordering

No strict global ordering is enforced; the server timestamps each message on receipt (via NTP-synced clocks) and clients display in that order — a deliberately simpler stance than trying to build a fully ordered log.

## Deep dive 6 — "last seen" without write amplification

- **Bad:** write to the DB on every heartbeat — at billions of users this is enormous write volume for a low-value field.
- **Great:** only persist a disconnect timestamp (conditional write, DynamoDB), and answer "is this user online right now" by querying their currently-assigned Chat Server via the Pub/Sub layer rather than a database read at all.

## Final architecture

L4 Load Balancer → Chat Servers (consistent-hash assigned) → DynamoDB (Chat, ChatParticipant, Message, Inbox, Client, LastSeen) + sharded Redis Pub/Sub (real-time delivery, sequence-numbered) + S3/CDN (media, presigned) + NTP-synchronized timestamping.

## Level expectations

- **Mid (E4):** ~80% breadth; clean API/data model, functional design, rough sense of the scaling problem; interviewer guides the deep dives.
- **Senior (E5):** ~60% breadth / 40% depth; speeds through the basics, discusses scaling/robustness specifics, articulates tradeoffs, proactively identifies problems.
- **Staff+ (L7+):** ~40% breadth / 60% depth; exceptional unprompted proactivity, advanced failure-mode analysis, comfortable discussing regionalization/cell-based architecture.
