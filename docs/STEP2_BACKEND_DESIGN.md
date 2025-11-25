# Step 2 – Backend Architecture Design

Date: 2025-11-25

## Goals

* Provide FastAPI + Socket.IO backend that satisfies the real-time proctoring spec.
* Keep all state in memory with clear ownership and lifecycle rules.
* Support screen-share fan-out to many supervisors via `aiortc`.

## High-level Components

1. **FastAPI App + Uvicorn** – serves REST endpoints for lobby/room metadata and hosts Socket.IO ASGI app.
2. **python-socketio AsyncServer** – handles lobby updates, signalling messages, and room membership notifications.
3. **RoomRegistry** – process-wide data structure storing rooms, participants, and WebRTC peer connections.
4. **WebRTCManager** – wraps `aiortc` to create/maintain RTCPeerConnections per student (upstream) and supervisor (downstream clones).

## Data Structures

```python
class Participant(BaseModel):
    session_id: str
    identity: str
    role: Literal["student", "supervisor"]
    joined_at: datetime

class StudentState(Participant):
    is_sharing: bool
    pc: RTCPeerConnection | None
    stream: MediaStreamTrack | None

class SupervisorState(Participant):
    is_host: bool

class Room(BaseModel):
    room_id: str
    title: str
    host_id: str
    created_at: datetime
    students: dict[str, StudentState]
    supervisors: dict[str, SupervisorState]
```

Additional helpers:

* `RoomRegistry` with methods: `list_rooms()`, `create_room()`, `join_room()`, `leave_room()`, `transfer_host()`, `terminate_room()`, `room_summary(room_id)`.
* `SessionIndex` map session_id → (room_id, role) to cleanly remove participants after disconnects.

### Supervisor Roles

* **Host Supervisor (방장)** – first supervisor in a room; can create/terminate rooms, admit students beyond lobby, and optionally transfer host to another supervisor.
* **Guest Supervisor (참관)** – read-only monitor; can join rooms, view streams, but cannot terminate or modify room state.
* Host identity is stored in both `room.host_id` and `SupervisorState.is_host`. Socket events surface updates so clients enable/disable privileged UI accordingly.

## REST API Surface

| Method | Path | Summary |
| --- | --- | --- |
| `GET /health` | readiness check |
| `GET /rooms` | list lobby cards (`{roomId,title,studentCount,supervisorCount}`) |
| `POST /rooms` (host supervisor) | create room (`title`) and auto-subscribe host |
| `POST /rooms/{roomId}/join` | join by role; returns socket/token metadata |
| `POST /rooms/{roomId}/leave` | explicit leave (optional; socket disconnect also handles) |
| `POST /rooms/{roomId}/terminate` | host-only action to close room |
| `POST /rooms/{roomId}/host/transfer` | current host grants host privileges to another supervisor (fallback) |

Authentication is lightweight: REST accepts `{sessionId, identity, role}` from client; session IDs come from Socket.IO connection IDs to keep state consistent. For MVP we trust client-provided role but enforce constraints (e.g., students limited to six) server-side.

### NAT Traversal & ICE

* The Python media server must run on a public IP so both students and supervisors can reach it directly over UDP (the server acts as the rendezvous point and RTP relay).
* We will rely on public STUN to obtain server-reflexive candidates; TURN is optional and can be skipped initially.
* Recommended ICE configuration (shared with browsers and `aiortc`):

```python
ICE_SERVERS = [
    {"urls": ["stun:stun.l.google.com:19302"]},
]
```

* If stricter firewall environments appear later, we can add a TURN server, but the baseline deployment assumes the backend is internet-accessible and the free Google STUN is sufficient.

## Socket.IO Namespaces & Events

Single namespace `/rtc`.

### Client → Server events

* `lobby:subscribe` – join lobby updates room.
* `room:subscribe` – join Socket.IO room for specific `roomId`.
* `room:signal` – generic signalling payload (type, to, description, candidate). Students send offer; supervisors send answer; server relays or uses aiortc.
* `room:share:start` / `room:share:stop` – inform server of sharing state (sync with WebRTC manager).
* `room:terminate` – host requests termination.
* `room:host:transfer` – host designates another supervisor as host (optional fallback).

### Server → Client events

* `lobby:update` – broadcast list of rooms when changed.
* `room:participants` – current participant metadata for grid.
* `room:student:update` – specific student share state changes.
* `room:host:update` – announces which supervisor currently has host privileges.
* `room:terminated` – instruct clients to exit room.
* `room:signal` – relay ICE/sdp messages back to peers as needed.

## WebRTC Signalling Flow

1. Student opens room, clicks share:
    * Browser obtains MediaStream.
    * Student emits `room:share:start` with SDP offer.
    * Server creates `aiortc.RTCPeerConnection`, adds the incoming track as an upstream publisher, and stores both the connection and its `MediaStreamTrack` on `StudentState`.
    * Server responds with answer; student connection becomes the **single RTP ingress** for that identity.
2. Supervisor subscribes:
    * For each sharing student, `WebRTCManager` creates a downstream `RTCPeerConnection` for the supervisor.
    * The stored upstream track is **cloned** (via `track.clone()`) and attached to the supervisor PC; aiortc handles packet forwarding from the original RTP stream.
    * ICE candidates forwarded both ways through `room:signal`.
3. Hover zoom is implemented purely client-side CSS; backend only ensures remote streams delivered.

### RTP Relay Responsibilities

* Students never receive tracks; their upstream PC only **publishes** RTP to the server.
* Supervisors never publish; their PC only **subscribes** to cloned tracks. Host supervisors and guest supervisors use the same media pipeline; the only difference is control permissions enforced at the signalling layer.
* Server acts as an SFU-lite relay:
    * Maintains exactly one upstream PC per sharing student.
    * Fans out the buffered RTP packets to N downstream supervisors by attaching cloned tracks to each supervisor PC.
    * When a supervisor disconnects, its downstream PC is closed without affecting other supervisors or the student's upstream.
* This architecture keeps bandwidth per student at 1× uplink regardless of supervisor count; server handles duplication.

## Concurrency & Cleanup

* `RoomRegistry` guarded by `asyncio.Lock`.
* On Socket.IO `disconnect`, lookup session in `SessionIndex`, remove from room, release WebRTC peers.
* Terminate room: close PCs, pop room, broadcast updates.

## Implementation Tasks (feeds Step 3)

1. Configure FastAPI + Socket.IO (ASGI lifespan, uvicorn entrypoint).
2. Implement `RoomRegistry` module with thread-safe operations.
3. Build REST endpoints referencing registry and emitting lobby events.
4. Implement Socket.IO event handlers for lobby and rooms.
5. Create `webrtc.py` with helpers for student upstream and supervisor downstream handling (offer/answer, candidate exchange).
6. Add periodic cleanup (optional) for stale rooms.
7. Write smoke tests (if time) for registry logic.
