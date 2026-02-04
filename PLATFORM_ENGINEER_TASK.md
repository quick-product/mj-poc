# Durable Objects - Migrate or not Migrate, that is the question.

## Time Allocation: 3 hours

## Background

We are currently using **Cloudflare Durable Objects** for real-time meeting processing. Durable Objects provide:

- **Fast cold start** (~50-100ms spin-up time)
- **Dedicated memory per instance** (128MB heap RAM per Durable Object)
- **In-memory state** within a single instance lifetime
- **WebSocket support** with persistent connections
- **Simple addressing** via a unique global ID
- **Automatic scaling**

We are considering migrating to **AWS** and need an architecture that replicates these characteristics.

---

## Current Interface (Cloudflare)

```typescript
// Get a Durable Object stub by session name
const doId = env.STT_ADAPTER.idFromName(sessionName)
const stub = env.STT_ADAPTER.get(doId)

// Forward any HTTP/WebSocket request to the DO
const response = await stub.fetch(request)
```

The Durable Object itself:
- Handles HTTP requests via `fetch(request: Request)`
- Manages WebSocket connections via `webSocketMessage()`, `webSocketClose()`, `webSocketError()`
- Maintains in-memory state throughout the session lifetime

---

## What the Durable Object Does

The STT (Speech-to-Text) Adapter Durable Object handles **live meeting transcription**:

1. **Receives WebRTC audio** from clients via SFU
2. **Maintains WebSocket connections** for:
   - SFU audio input stream
   - Transcription output stream to clients
3. **Processes audio** - resampling, buffering, forwarding to a transcription service (e.g. Soniox)
4. **Manages meeting lifecycle** - connection, recording, finalization, cleanup
5. **Maintains in-memory state** - session data, audio buffers, connection tracking
6. **Schedules timers** - inactivity timeouts, finalization deadlines, cleanup

---

## Requirements for AWS Architecture

### Core Requirements

1. **Sub-second instance startup** (ideally <500ms, max 1 second)
2. **Dedicated memory per instance** - each DO equivalent gets its own isolated memory
3. **In-memory state** - state maintained in RAM throughout the session lifetime
4. **WebSocket support** with persistent connections
5. **Simple API interface** - pass session ID, get back a "stub" to communicate with
6. **Horizontal scaling** - support 2,000+ simultaneous meetings
7. **Cost efficiency** - scale down gracefully when demand is low (e.g. 50 simultaneous meetings)
8. **Garbage collection** - clean up resources and release memory after meeting ends
9. **Code deployment** - clear path for deploying/updating the DO code

### Out of Scope

- Hibernation (instances stay active during meetings)
- State persistence to disk/database (in-memory only)
- Multi-region replication

---

## Tasks

### Task 1: Architecture Design

**Create a technical architecture diagram** that shows:

1. **AWS Services** to be used and how they interconnect
2. **Request flow** from API to the Durable Object equivalent
3. **Memory isolation** - how each instance gets dedicated RAM (~128MB heap per session)
4. **WebSocket handling** - how WS connections are routed and maintained
5. **Scaling strategy** for both high demand (2,000 meetings) and low demand (50 meetings)
6. **Garbage collection** mechanism for cleaning up after meetings end and releasing memory
7. **Deployment strategy** - how DO code is deployed and updated

**Deliverable**: Architecture diagram with written explanation of:
- Service choices and rationale
- How memory isolation is achieved per session
- Tradeoffs considered
- Estimated costs at different scales (50 vs 2,000 meetings)
- Cold start time estimates

### Task 2: Infrastructure Implementation

**Implement the core infrastructure** using AWS CDK, Pulumi or any other IaC tool of your choice.

Build a **simple proof of concept** that demonstrates:

1. **Web Audio Recorder** - Simple web page that:
   - Captures microphone audio using Web Audio API
   - Sends PCM audio buffers to the AWS "Durable Object"
   
2. **AWS Durable Object** that:
   - Receives audio buffers via WebSocket
   - Stores received audio to a file system (S3, EFS, or local)
   - Demonstrates the session-based addressing pattern

**Deliverable**: 
- IaC file in a git repository
- Simple HTML/JS page for audio capture
- README with deployment instructions

**Simplifications:**
- Instead of webrtc use websockets to stream to DO (webrtc is too complex for this task)

### Task 3: Transcription Pipeline (Optional - Bonus Points)

**Extend Task 2** to include real-time transcription:

1. **Stream audio to Soniox** (we'll provide API credentials)
2. **Receive transcription results** in real-time
3. **Stream** transcription results back the client
4. **Visualize** transcription results in a browser

**Deliverable**:
- Working transcription pipeline

GLHF!
