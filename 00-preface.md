# Durable Objects to AWS Migration: Preface

**Status:** Draft  
**Date:** 2025-02-04  
**Time Allocation:** 3 hours  

---

## 1. Problem Statement

We are migrating from Cloudflare Durable Objects to AWS for real-time meeting transcription. The system requires stateful, session-isolated workers that receive audio streams via WebSocket, buffer them in memory, forward to an internal transcription API, and stream results back to clients.

### What Durable Objects Provide

| Capability | Cloudflare DO | AWS Requirement |
|------------|---------------|-----------------|
| Cold start | 50-100ms | Sub-500ms (max 1s) |
| Memory per instance | 128MB heap | 128MB dedicated per session |
| State isolation | Automatic per ID | Must replicate |
| WebSocket support | Native | Required |
| Addressing | `idFromName(session)` | Session-based routing |
| Scaling | Automatic | Autoscale up and down |
| Hibernation | Supported | Scale to minimal when idle |
| Garbage collection | Automatic | Must implement |

### Current Interface (Cloudflare)

```javascript
// Get a Durable Object stub by session name
const doId = env.STT_ADAPTER.idFromName(sessionName)
const stub = env.STT_ADAPTER.get(doId)

// Forward any HTTP/WebSocket request to the DO
const response = await stub.fetch(request)
```

### What the Durable Object Does

The STT (Speech-to-Text) Adapter handles live meeting transcription:

1. Receives audio from clients via WebSocket
2. Buffers and processes audio (resampling, chunking)
3. Forwards to internal transcription API (Soniox)
4. Streams transcription results back to clients
5. Manages session lifecycle (connection, recording, finalization, cleanup)
6. Maintains in-memory state throughout session lifetime

---

## 2. Requirements

### From Specification

| Requirement | Detail |
|-------------|--------|
| Instance startup | Sub-500ms, max 1 second |
| Memory isolation | 128MB dedicated heap per session |
| State model | In-memory only during session |
| WebSocket | Persistent connections required |
| Horizontal scaling | Autoscale based on demand |
| Scale down | Hibernate/scale to minimal when idle |
| Garbage collection | Clean up resources after meeting ends |
| Code deployment | Clear path for updates |

### Out of Scope (per specification)

- State persistence to disk/database (in-memory only)
- Multi-region replication
- Hibernation mid-session (instances stay active during meetings)

---

## 3. Assumptions

The following are derived from clarifications and are **not in the original requirements document**:

### Infrastructure

1. **Single region** for POC (e.g., us-east-1); multi-region is future scope
2. **No existing AWS infrastructure** - starting fresh
3. **Terraform** for Infrastructure as Code
4. **Observability** - CloudWatch native, with option to export to Grafana

### Authentication

5. **JWT authentication** - required but not trivial; defer detailed implementation to later phase

### Clients

6. **Multiple client platforms** (web, mobile) but we control audio format
7. **Audio format** - consistent PCM format controlled by client apps (e.g., 16-bit, 16kHz mono)

### Sessions

8. **Meeting duration** - no upper limit
9. **Session ID generation** - API generates; clients do not provide
10. **No session query** - purely key-value lookup by meeting_id; no "list all sessions"

### Data Handling

11. **Audio retention** - in-memory until transcription completes; offload to S3 under memory pressure; cleanup after success
12. **Transcription output** - stored in KV/DB by meeting_id
13. **Downstream consumers** - access results via meeting_id from shared store; event published to SQS/EventBridge on completion

### Failure Handling

14. **POC failure model** - worker crash means client uploads full audio file at meeting end as fallback
15. **Production failure model** - worker recovery with checkpointing (future scope)

### Protocol

16. **WebSocket for POC** - sufficient for transcription latency requirements
17. **WebRTC/SFU for future** - required only when sub-100ms latency needed

---

## 4. Constraints

### Hard Constraints

- Sub-1 second cold start for session handling
- 128MB memory isolation per session
- WebSocket support for audio streaming
- Autoscaling (no fixed capacity limits)
- Scale to minimal during low/no demand

### Soft Constraints (POC trade-offs acceptable)

- Direct IP routing acceptable (no load balancer in audio path)
- Simple failure model (client fallback upload)
- Single region deployment

---

## 5. Document Structure

| Document | Purpose |
|----------|---------|
| **00-preface.md** (this document) | Problem statement, requirements, assumptions |
| **01-poc-architecture.md** | Minimal viable architecture for POC |
| **02-production-architecture.md** | Full options analysis, scaling, future roadmap |

---

## 6. References

- [Cloudflare Durable Objects Documentation](https://developers.cloudflare.com/durable-objects/)
- [AWS Real-Time Communication Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/real-time-communication-on-aws/real-time-communication-on-aws.html)
- [AWS WebSocket API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html)
- [AWS Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
