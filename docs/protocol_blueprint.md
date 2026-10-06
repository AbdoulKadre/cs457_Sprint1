# Application Protocol Blueprint — Tic-Tac-Toe over TCP (v1.0)

**Course:** Network Programming — Sprint 1  
**Student:** Abdoul Kadre Tiemtore  
**Document:** `protocol_blueprint.md`  

---

## 1. Overview

This document fully specifies the custom application layer protocol (**TTT-1**) used by the client server Tic-Tac-Toe game. TTT-1 runs over TCP and defines framing rule, exact message schemas for all client server communications, and the connection termination semantics used by the game state machine.

Design principles:

1. Every byte on the wire is either a message or a framing byte; the receiver can always recover message boundaries.
2. Every field has a fixed type, an allowed range, and documented semantics.
3. Every legal game event has a defined message or a defined transport level fallback.

---

## 2. Transport Layer


| Property             | Value                                                                         |
| -------------------- | ----------------------------------------------------------------------------- |
| Transport protocol   | TCP                                |
| Addressing           | Server listens on a configurable IPv4 address, default port `5555`            |
| Roles                | One server process; exactly two client connections per game room              |
| Serialization format | UTF-8 encoded JSON                                                            |
| Keep-alive           | None at application layer; TCP relies on OS keep alive + socket timeout  |


---

## 3. Framing Rule (Option A: Newline-Delimited JSON)

### 3.1 Rule

- Every message is a single JSON object, serialized compactly and encoded as UTF-8.
- Every message **must** be terminated by a single newline character `\n` (`0x0A`).
- JSON strings can never contain a raw `0x0A` byte: JSON encoders must escape newlines as `\n` inside strings, so `0x0A` is unambiguous as a delimiter.
- Maximum frame length: **4096 bytes** including the newline. Frames exceeding this limit are a protocol violation (server replies `ERROR: FRAME_TOO_LONG` and closes the connection).

### 3.2 Receiver algorithm 

The receiver accumulates bytes into a per connection buffer until a complete frame is available:

```
1. buf: bytes = b""
2. loop:
     chunk = sock.recv(4096)
     if chunk is empty (0 bytes):        # TCP EOF — peer closed cleanly
         signal FSM: CLIENT_DISCONNECTED ; close socket ; exit
     buf += chunk
     while b"\n" in buf:
         frame, buf = split buf at first b"\n"
         if len(frame) > 4096: ERROR FRAME_TOO_LONG; disconnect
         msg = json.loads(frame)          # on failure: ERROR MALFORMED_JSON
         dispatch(msg)
```

This handles coalescing and fragmentation.

### 3.3 Raw wire stream example

Two messages sent back to back, arriving coalesced in one `recv()`:

```
{"msg_type":"CONNECT","player_id":"Alice","timestamp":1727000000}\n{"msg_type":"MOVE","player_id":"Alice","payload":{"row":0,"col":2},"timestamp":1727000005}\n
```

Hex view of the delimiter (fragmented case — first `recv()` returns half a message):

```
recv #1: 7B 22 6D 73 67 5F 74 79 70 65 22 3A 22 4D 4F 56   {"msg_type":"MOV
recv #2: 45 22 2C 22 70 6C 61 79 65 72 5F 69 64 22 3A 22   E","player_id":"
...      ... 0A                                          ... \n
```

The receiver only deserializes when the `0A` byte arrives.

---

## 4. Envelope Schema (all messages)

Every message is a JSON object with this common envelope:


| Field       | Type    | Required          | Semantics                                                                                                                                   |
| ----------- | ------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `msg_type`  | string  | yes               | One of the 8 types in §5 (uppercase).                                                                                                       |
| `player_id` | string  | yes               | Alias chosen at CONNECT. Set by the server on server→client messages to address the target player. |
| `payload`   | object  | yes  | Type specific fields.                                                                                                                  |
| `timestamp` | integer | yes               | Unix epoch seconds when the message was created.                                                                                            |


Unknown fields must be ignored. A missing required field or a wrong type is a `MALFORMED_MESSAGE` error.

JSON Schema for the envelope:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TTT-1 Envelope",
  "type": "object",
  "required": ["msg_type", "player_id", "payload", "timestamp"],
  "properties": {
    "msg_type": { "enum": ["CONNECT", "LOBBY_WAIT", "GAME_START", "MOVE",
                            "STATE_UPDATE", "ERROR", "DISCONNECT", "GAME_OVER"] },
    "player_id": { "type": "string", "minLength": 1, "maxLength": 16,
                   "pattern": "^[\\x20-\\x7E]+$" },
    "payload": { "type": "object" },
    "timestamp": { "type": "integer", "minimum": 0 }
  }
}
```

---

## 5. Message Type Inventory


| #   | Message Type   | Direction        | Purpose                                                                                             |
| --- | -------------- | ---------------- | --------------------------------------------------------------------------------------------------- |
| 1   | `CONNECT`      | Client → Server  | Client requests to join the game room with a player alias.                                          |
| 2   | `LOBBY_WAIT`   | Server → Client  | Server notifies Player 1 that it is waiting for Player 2 to connect.                                |
| 3   | `GAME_START`   | Server → Clients | Server notifies both clients that the game has started; assigns roles. |
| 4   | `MOVE`         | Client → Server  | Active player submits move coordinates (`row`, `col`).                                       |
| 5   | `STATE_UPDATE` | Server → Clients | Server broadcasts updated board state, scores, and whose turn it is.                                |
| 6   | `ERROR`        | Server → Client  | Server notifies a client of an out-of-turn move, invalid coordinates, or a malformed message.       |
| 7   | `DISCONNECT`   | Client → Server  | Client notifies the server of an intentional departure/quit (forfeit).                              |
| 8   | `GAME_OVER`    | Server → Clients | Server broadcasts final outcome (winner / draw / forfeit) and final scores.                         |


---

## 6. Message Schemas

### 6.1 CONNECT (Client → Server)

```json
{ "msg_type": "CONNECT", "player_id": "Alice", "payload": {}, "timestamp": 1727000000 }
```


| Payload field | Type | Constraint         |
| ------------- | ---- | ------------------ |
| *(none)*      | —    | Empty object `{}`. |


Server behavior: first CONNECT → assigned Player 1 (X), receives `LOBBY_WAIT`. Second CONNECT → `GAME_START` broadcast to both.

### 6.2 LOBBY\_WAIT (Server → Client)

```json
{ "msg_type": "LOBBY_WAIT", "player_id": "Alice", "payload": { "role": "X", "position": 1 }, "timestamp": 1727000010 }
```


| Payload field | Type    | Constraint     |
| ------------- | ------- | -------------- |
| `role`        | string  | `"X"` or `"O"` |
| `position`    | integer | `1` or `2`     |


### 6.3 GAME\_START (Server → Clients)

```json
{ "msg_type": "GAME_START", "player_id": "Alice", "payload": { "role": "X", "opponent_id": "Bob", "board": ["........."], "first_turn": "Alice" }, "timestamp": 1727000020 }
```


| Payload field | Type               | Constraint                                                                      |
| ------------- | ------------------ | ------------------------------------------------------------------------------- |
| `role`        | string             | `"X"` or `"O"`                                                                  |
| `opponent_id` | string             | Alias of the opponent                                                           |
| `board`       | array of 3 strings | Each string exactly 3 chars from `{".", "X", "O"}`; `"..."` per row means empty |
| `first_turn`  | string             | `player_id` of the active player (always X)                                     |


### 6.4 MOVE (Client → Server)

```json
{ "msg_type": "MOVE", "player_id": "Alice", "payload": { "row": 0, "col": 2 }, "timestamp": 1727000005 }
```


| Payload field | Type    | Constraint    |
| ------------- | ------- | ------------- |
| `row`         | integer | `0 ≤ row ≤ 2` |
| `col`         | integer | `0 ≤ col ≤ 2` |


### 6.5 STATE\_UPDATE (Server → Clients)

```json
{ "msg_type": "STATE_UPDATE", "player_id": "Bob", "payload": { "board": ["OX.", "...", "..."], "active_turn": "Bob", "scores": { "Alice": 0, "Bob": 0 } }, "timestamp": 1727000010 }
```


| Payload field | Type               | Constraint                                 |
| ------------- | ------------------ | ------------------------------------------ |
| `board`       | array of 3 strings | 3 chars each from `{".", "X", "O"}`        |
| `active_turn` | string             | `player_id` of the player whose turn it is |
| `scores`      | object             | Map of `player_id` → integer score         |


### 6.6 ERROR (Server → Client)

```json
{ "msg_type": "ERROR", "player_id": "Alice", "payload": { "code": "OUT_OF_TURN", "detail": "It is Bob's turn." }, "timestamp": 1727000015 }
```


| Payload field | Type   | Constraint                               |
| ------------- | ------ | ---------------------------------------- |
| `code`        | string | One of the codes below                   |
| `detail`      | string | Human-readable explanation (≤ 200 chars) |


**Error codes:**


| Code                | Meaning                                           |
| ------------------- | ------------------------------------------------- |
| `MALFORMED_JSON`    | Frame is not valid JSON                           |
| `MALFORMED_MESSAGE` | JSON valid, but schema validation failed          |
| `UNKNOWN_TYPE`      | `msg_type` not in the inventory                   |
| `OUT_OF_TURN`       | Move received from the inactive player            |
| `INVALID_MOVE`      | Coordinates out of range or cell already occupied |
| `NOT_CONNECTED`     | MOVE/DISCONNECT received before CONNECT           |
| `FRAME_TOO_LONG`    | Frame exceeded 4096 bytes                         |
| `ROOM_FULL`         | Third CONNECT attempt while a game is in progress |


### 6.7 DISCONNECT (Client → Server)

```json
{ "msg_type": "DISCONNECT", "player_id": "Alice", "payload": { "reason": "forfeit" }, "timestamp": 1727000030 }
```


| Payload field | Type   | Constraint              |
| ------------- | ------ | ----------------------- |
| `reason`      | string | `"forfeit"` or `"quit"` |


### 6.8 GAME\_OVER (Server → Clients)

```json
{ "msg_type": "GAME_OVER", "player_id": "Bob", "payload": { "outcome": "WIN", "winner": "Alice", "reason": "LINE_COMPLETE", "final_board": ["XOX", "OX.", "X.."], "final_scores": { "Alice": 1, "Bob": 0 } }, "timestamp": 1727000040 }
```


| Payload field  | Type               | Constraint                                        |
| -------------- | ------------------ | ------------------------------------------------- |
| `outcome`      | string             | `"WIN"`, `"DRAW"`, or `"FORFEIT"`                 |
| `winner`       | string or null     | `player_id` of the winner, `null` on draw         |
| `reason`       | string             | `"LINE_COMPLETE"`, `"BOARD_FULL"`, or `"FORFEIT"` |
| `final_board`  | array of 3 strings | Final board state                                 |
| `final_scores` | object             | `player_id` → integer score                       |


---

## 8. Connection Termination &amp; Socket Lifecycle

### 8.1 Three termination paths

1. **Application layer disconnect (`DISCONNECT` message)** — a client deliberately quits. 
2. **Transport layer teardown (TCP FIN)** — a client process closes its socket or exits normally without sending `DISCONNECT`. 
3. **Abrupt termination (TCP RST)** — the client process is killed (`kill -9`), the host loses power, or the network link drops. 

### 8.2 Server side dispatchbloop handling 

```python
try:
    data = recv_frame(sock, buf)          # returns None on clean EOF (b"")
    if data is None:                       # TCP EOF: peer closed cleanly
        trigger_state_transition("CLIENT_DISCONNECTED")
except TimeoutError:
    logger.warning("Peer stopped responding (timeout).")
    trigger_state_transition("CLIENT_DISCONNECTED")
except (ConnectionResetError, BrokenPipeError, ConnectionAbortedError) as e:
    logger.warning(f"Connection lost abruptly: {e}")
    trigger_state_transition("CLIENT_DISCONNECTED")
```

### 8.3 Send rule

All sends use `sendall()`. `BrokenPipeError` or `ConnectionResetError` on send is caught, logged, and routed to the same `CLIENT_DISCONNECTED` transition — never allowed to crash the dispatch loop.

### 8.4 Resource cleanup checklist (executed on every termination path)

- Remove the connection from the room registry
- Broadcast `GAME_OVER` to the surviving opponent, if any
- `close()` both sockets, discard stream buffers
- FSM transition: current state → `WAITING_FOR_PLAYERS`

---

## 9. Protocol Violations &amp; Recovery


| Violation                           | Detection              | Recovery                                                            |
| ----------------------------------- | ---------------------- | ------------------------------------------------------------------- |
| Invalid JSON frame                  | `json.loads` exception | Send `ERROR MALFORMED_JSON`; keep connection (single-strike policy) |
| Schema violation                    | field validation       | Send `ERROR MALFORMED_MESSAGE`; keep connection                     |
| Frame &gt; 4096 bytes               | length check           | Send `ERROR FRAME_TOO_LONG`; close connection                       |
| Message before CONNECT              | state check            | Send `ERROR NOT_CONNECTED`                                          |
| Move out of turn                    | turn check             | Send `ERROR OUT_OF_TURN`; game continues                            |
| Invalid coordinates / occupied cell | board check            | Send `ERROR INVALID_MOVE`; same player retries                      |
| Third CONNECT while room full       | slot check             | Send `ERROR ROOM_FULL`; close connection                            |


---

## 10. Design Rationale

- Newline delimited JSON over binary length-prefix.
- Explicit error codes instead of closing connections.
- Uniform envelope (`msg_type`/`player_id`/`payload`/`timestamp`) simplifies the dispatcher into a single `switch (msg_type)` and makes every frame parseable by one schema.

*Companion documents: `fsm_specification.md` and, `ai_prompts.md`*
