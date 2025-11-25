# HTTP Test Scenarios

This document lists manual HTTP sequences you can replay (e.g., with `curl`, REST Client, or Postman) to exercise every REST endpoint exposed by the backend.

> Replace placeholder values such as `ROOM_ID` or `SESSION` with actual IDs returned from earlier calls.

1. **Health check**

```bash
curl -X GET http://localhost:8010/health
```

2. **List rooms**

```bash
curl -X GET http://localhost:8010/rooms
```

3. **Create room (host supervisor)**

```bash
curl -X POST http://localhost:8010/rooms \
  -H "Content-Type: application/json" \
  -d '{"title": "선형대수학 중간고사", "hostSessionId": "sup-host-1", "identity": "감독 김"}'
```

Record the returned `roomId`.

4. **Join room as student**

```bash
curl -X POST http://localhost:8010/rooms/$ROOM_ID/join \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "stu-001", "identity": "20230001 김학생", "role": "student"}'
```

5. **Join room as guest supervisor**

```bash
curl -X POST http://localhost:8010/rooms/$ROOM_ID/join \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "sup-guest-1", "identity": "감독 박", "role": "supervisor"}'
```

6. **Transfer host privileges**

```bash
curl -X POST http://localhost:8010/rooms/$ROOM_ID/host/transfer \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "sup-host-1", "targetSessionId": "sup-guest-1"}'
```

7. **Leave room**

```bash
curl -X POST http://localhost:8010/rooms/$ROOM_ID/leave \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "stu-001"}'
```

8. **Terminate room (host only)**

```bash
curl -X POST http://localhost:8010/rooms/$ROOM_ID/terminate \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "sup-host-1"}'
```

After termination, hitting `/rooms` again should show an empty list.
