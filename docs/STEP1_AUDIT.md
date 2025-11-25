# Step 1 – Scaffolding Audit

Date: 2025-11-25

## Backend (python/FastAPI)

Current state:

* `backend/main.py` contains only a `main()` function that prints `"Hello from backend!"`.
* `pyproject.toml` defines no dependencies (FastAPI, uvicorn, python-socketio, aiortc are missing).
* `uv.lock` and `README.md` exist but are empty placeholder content.

Missing vs. spec:

* FastAPI application with HTTP endpoints (`/health`, lobby/rooms, etc.).
* Socket.IO (or similar) server for realtime lobby + signalling.
* WebRTC peer management using `aiortc`, track fan-out, and screen-share handling.
* In-memory room registry with host/student/supervisor tracking.
* Room lifecycle logic (create/join/leave/end) and enforcement of capacity/role rules.
* Supervisor vs. student permissions, host-only room termination.
* Any tests or dev tooling (uv scripts).

## Frontend (React/Vite)

Current state:

* No `frontend` directory yet.

Missing vs. spec:

* Vite project scaffold with React, routing, and CSS framework (Tailwind or CSS Modules).
* Pages: landing (`/`), lobby (`/lobby`), room (`/room/:roomId`) with required UI widgets.
* State management for session identity (role/username) using session storage.
* WebSocket client for lobby updates and room membership.
* WebRTC client logic for screen sharing, receiving remote tracks, and hover zoom effects.
* Host controls (end room) and supervisor multi-stream monitoring grid.

## Next Actions

* Proceed to Step 2 (backend design) now that gaps are cataloged.
