# “WhatsApp/Slack-like” chat system design that works from 10K to 100M+ users, supports 1:1 + groups, multi-device, offline, search, attachments, and high availability.

Requirements
Core features

Send/receive messages in real time (1:1, group/channel)

Delivery states: sent → delivered → read

Offline support + catch-up

Media/file sharing

Push notifications

Search (at least within a conversation; Slack-style global optional)

Presence/typing indicators (best-effort)

Non-functional

Low latency: <200ms typical send-to-receive

Massive fanout for big groups

High availability, multi-region

Security (auth, rate limits, abuse prevention; optional E2EE)

High-level architecture

Clients (mobile/web/desktop)
↕ (WebSocket/gRPC stream + HTTPS for uploads/downloads)
API Gateway / Edge (TLS termination, auth, rate-limit)
↕
Real-time Gateway (Connection Service) (maintains WebSockets, presence)
↕
Chat Service (Message Ingest + Routing)
↕
Durable Log / Queue (Kafka/Pulsar)
↕
Message Store (NoSQL) + Metadata Store (SQL/NoSQL)
↕
Fanout/Delivery Workers (deliver to online devices, schedule push)
↕
Push Service (APNS/FCM) + Notification Service
↕
Search Index (Elastic/OpenSearch)
↕
Media Service (pre-signed URLs) + Object Storage (S3/Blob) + CDN

Key services and responsibilities
1) Connection Service (WebSocket Gateway)

Keeps persistent connections: userId -> deviceId -> connectionId

Publishes presence state (online/last seen) with TTL

Subscribes to “deliver message to user/device” topics

Handles “typing”, read receipts (best-effort)

Scale trick: shard by userId or deviceId with consistent hashing so reconnects tend to land on same shard, but don’t rely on it.

2) Chat Service (Message Ingest)

Responsibilities:

Validate auth + membership (can user post in channel?)

Assign server IDs (monotonic per conversation if needed)

Persist message (durable write)

Emit events to log (Kafka): MessageCreated, MessageEdited, MessageDeleted

Write path:
Client → Chat Service → Message Store (write) → Kafka (event)

Make the DB write the source of truth; Kafka ensures scalable downstream processing.

3) Fanout + Delivery

Two modes:

A) Fanout-on-write (Slack-like channels, many online users)

On message create, compute recipients and enqueue deliveries

Good for fast inbox updates, but heavy for huge groups

B) Fanout-on-read (WhatsApp-like, simpler storage)

Store message once per conversation

Each user maintains a “cursor” (last seen message)

Client fetches missing messages on reconnect

Best practical approach: hybrid

Small/medium groups: fanout-on-write to per-user “inbox”

Very large groups/channels: fanout-on-read + caching + online delivery only

Online delivery: push via Connection Service if recipient device connected.
Offline: push notification + they pull messages when online.

Data model (simplified)
Conversations

Conversation { id, type (dm/group/channel), createdAt, ... }

Membership { conversationId, userId, role, joinedAt, lastReadMessageId, muted }

Messages

Store partitioned by conversation:

Message { conversationId (partition key), messageId (sort key), senderId, ts, body, attachments, edited, deleted }

Why partition by conversation? Fast “load last N messages” and ordered pagination.

Delivery receipts (don’t store per-recipient at huge scale)

For DMs/small groups: store per-user receipt state

For large channels: store only per-user read cursor + maybe per-message aggregate counts

Presence:

Redis-like TTL: presence:userId -> online/lastSeen

Ordering, IDs, and consistency

Use server-assigned messageId (e.g., time-based UUID/ULID) for global uniqueness.

If strict ordering per conversation matters:

Use DB sort key + server timestamp

Or allocate a per-conversation sequence via a lightweight sequencer (careful: bottleneck)

In practice: “mostly ordered” is fine, clients reconcile by (ts, messageId).

Consistency:

Messages: strong within a conversation partition write

Presence/typing: eventual/best-effort

Multi-device sync

Treat each device as a first-class subscriber.

Maintain Device { userId, deviceId, pushToken, lastSyncTs }

When a message arrives:

deliver to all online devices

update per-device sync cursor

On reconnect: client calls Sync(sinceCursor) and pulls missing messages.

Media/files

Upload via pre-signed URL to object store

Store message with attachment metadata (URL, size, mime, checksum)

Serve downloads via CDN

Virus scan / content moderation pipeline async

Search

Two levels:

Per-conversation search: index messages by (conversationId, ts, text)

Global search (Slack-like): index by (workspaceId/orgId, conversationId, userId perms)

Indexing pipeline:

MessageCreated → Search Worker → OpenSearch/Elastic

Apply ACL filtering at query time (or index per workspace)

Notifications

If recipient offline, enqueue push notification (APNS/FCM)

Throttle/mute rules, mention detection (@user, @channel)

“Notification collapse key” to avoid spamming

Security

Auth: OAuth2/JWT

Rate limits: per user + per IP + per conversation

Abuse: spam detection, link scanning, file limits

Encryption:

Transport: TLS always

At rest: encrypted disks/fields

Optional E2EE (WhatsApp): clients encrypt payloads; server stores ciphertext; receipts/presence stay server-visible unless you go full private mode

Scaling and partitioning strategy

Connection Service: scale horizontally; shard by user/device

Kafka topics: partition by conversationId for ordered processing per conversation

Message Store: partition by conversationId (hot partitions can happen in huge channels)

For huge channels: split by (conversationId, timeBucket) or adopt fanout-on-read + caching

Caches:

recent messages per conversation

membership lists (very important for routing)

Reliability patterns

Idempotency keys on send (clientMsgId) to avoid duplicates on retry

Exactly-once not required; aim for at-least-once + de-dupe on client/server

Store-and-forward: message is “sent” once durable write succeeds

Retry queues + DLQ for push and fanout

Backpressure: shed typing/presence first, then non-critical notifications

Observability

Tracing across gateway → chat → kafka → delivery

Metrics: p50/p95 send latency, delivery lag, WS connected users, push success rate

Audit logs for moderation/admin actions

A concrete API sketch

POST /messages { conversationId, clientMsgId, body, attachments }

GET /conversations/{id}/messages?before=&limit=

POST /conversations/{id}/read { lastReadMessageId }

GET /sync?cursor=

WebSocket events: message.new, message.edit, receipt.read, presence, typing
