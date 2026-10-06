# AI Prompting &amp; Constraint

**Course:** Network Programming — Sprint 1  
**Student:** Abdoul Kadre Tiemtore  
**Document:** `ai_prompts.md`  

---

## 1. Policy &amp; Goal

This is to constrain code generation as every AI session is anchored to the exact schemas in `protocol_blueprint.md`, and the AI is required to produce code that round-trips against those schemas byte-for-byte. AI output is treated as a candidate implementation, never as a specification source.

Constraint ladder applied to every prompt:

1. Paste the exact JSON schema before asking for code.
2. The framing rule is restated in the prompt itself, so the model cannot forget it.
3. Require specific function signatures; the AI fills only the bodies.
4. The AI must include unit tests proving wire format is conform.

---

## 2. System Prompt (used for all AI coding sessions)

> **ROLE:** You are a Python network-programming assistant implementing one function at a time for a TCP Tic-Tac-Toe game. You do NOT design the protocol. The protocol is FIXED and provided below; deviating from it is a failure.
>
> **HARD CONSTRAINTS:**
>
> 1. Transport: TCP. Serialization: UTF-8 JSON objects.
> 2. Framing: every message is a single compact JSON object terminated by the byte `0x0A` (newline). Never send a message without the trailing newline. Never assume one `recv()` equals one message — TCP coalesces and fragments frames; you MUST buffer until a complete newline terminated frame is available.
> 3. Envelope: every message has exactly the fields `msg_type`, `player_id`, `payload`, `timestamp`. No extra fields unless the schema for that `msg_type` defines them.
> 4. Message types are ONLY: CONNECT, LOBBY\_WAIT, GAME\_START, MOVE, STATE\_UPDATE, ERROR, DISCONNECT, GAME\_OVER — with the exact payload fields given in the provided schema excerpt. Do not invent new message types or payload fields.
> 5. EOF rule: `recv()` returning `b""` means the peer closed cleanly (EOF) — never loop on it; trigger the `CLIENT_DISCONNECTED` state transition and close the socket.
> 6. Exceptions: dispatch loops MUST catch `ConnectionResetError`, `BrokenPipeError`, `ConnectionAbortedError`, and `TimeoutError`, and route all of them to `CLIENT_DISCONNECTED`. No exception may crash the server loop.
> 7. Board encoding: 3×3 grid; `MOVE` payload has integer `row`, `col`, each `0..2`; board rows are 3-char strings over `{".", "X", "O"}`.
> 8. Output style: produce ONLY the requested function. Include a short docstring stating which schema clause it implements. Include pytest unit tests using the literal wire examples from the spec. Do not add error-handling, retry logic, or features not asked for.

---

## 3. Task Prompts

### 3.1 Frame encoder

> Using the envelope schema and the MOVE payload schema below, implement exactly:
>
> ```python
> def encode_message(msg_type: str, player_id: str, payload: dict, timestamp: int) -> bytes:
>     """Return the complete wire frame for one message, including the trailing b"\n"."""
> ```
>
> ```json
> { "msg_type": "MOVE", "player_id": "Player_1", "payload": { "row": 0, "col": 2 }, "timestamp": 1727000000 }
> ```
>
> Constraints: compact JSON (`separators=(',', ':')`), UTF-8, key order preserved as written in the schema, and the result MUST satisfy: `encode_message("MOVE", "Player_1", {"row":0,"col":2}, 1727000000)`.

### 3.2 Frame decoder 

> Implement the buffered receiver required by the framing rule:
>
> ```python
> class FrameReader:
>     def __init__(self, sock: socket.socket): ...
>     def read_frame(self) -> dict | None:
>         """Return the next parsed message dict.
>         Return None on clean EOF (recv() == b"") — do NOT loop on EOF.
>         Raise FrameTooLongError if a frame exceeds 4096 bytes.
>         Raise ConnectionResetError/BrokenPipeError/TimeoutError upward unchanged."""
> ```
>
> Constraints: never call `recv(1)`; accumulate into a buffer and split on `b"\n"`. Malformed JSON inside a complete frame must NOT close the connection — return a sentinel or raise `MalformedFrameError` so the caller can send `ERROR MALFORMED_JSON`.

### 3.3 FSM dispatcher

> Implement the server-side FSM from `fsm_specification.md` with this exact interface:
>
> ```python
> class GameFSM:
>     def handle(self, event: str, ctx: dict) -> list[dict]:
>         """event: one of CONNECT, MOVE, DISCONNECT, CLIENT_DISCONNECTED, ROOM_FULL_RETRY.
>         ctx: {'player_id':..., 'row':..., 'col':...} as applicable.
>         Returns the list of outbound messages (envelope dicts) that the caller must send."""
> ```
>
> Constraints: the only states are INIT, WAITING\_FOR\_PLAYERS, GAME\_START, PLAYER\_TURN, EVALUATE\_MOVE, GAME\_OVER; exactly the transition table from `fsm_specification.md` §4, no extra states. An out-of-turn MOVE must return `[{"msg_type":"ERROR","payload":{"code":"OUT_OF_TURN",...}}]` and NOT change state. Any `CLIENT_DISCONNECTED` during PLAYER\_TURN must produce a `GAME_OVER` with `outcome:"FORFEIT"` for the other player. 


### 3.4 Disconnect-handling prompt 
> Implement the server's permclient dispatch loop:
>
> ```python
> def serve_client(sock: socket.socket, fsm: GameFSM) -> None: ...
> ```
>
> Constraints: The `FrameReader` from 3.2. `read_frame() is None` (clean EOF) MUST trigger `CLIENT_DISCONNECTED`. Catch `(ConnectionResetError, BrokenPipeError, ConnectionAbortedError)` and 30 s `TimeoutError` and route them to the same transition, logging at WARNING. The loop must never let an exception propagate. After any disconnect path: remove the client from the room registry, notify the opponent with GAME\_OVER(FORFEIT), close the socket. 


---

## 4. Verification Workflow 

1. Paste the relevant schema + wire example into the session before any code request.
2. The AI must include a test asserting the literal wire bytes of `protocol_blueprint.md`.
3. The AI must include tests for malformed JSON, out of turn moves, bad coordinates, occupied cell, and oversized frames, each expecting the exact `ERROR`.
4. Field names match the schema; framing bytes present; EOF checked before use; all four exceptions caught; no invented message types; FSM states match the diagram.

---

## 5. Patterns Explicitly Banned from AI Output


| Anti-pattern                                                                      | Spec clause that forbids it         |
| --------------------------------------------------------------------------------- | ----------------------------------- |
| `sock.recv(1024).decode()` treated as one complete message                        | Framing rule                      |
| Ignoring `recv()` returning `b""`                              | EOF rule                        |
| Invented fields (`"type"` instead of `"msg_type"`, `"move"` instead of `payload`) | Envelope                          |
| Closing the connection on malformed JSON                                          | Violations table   |
| Bare `except:` that swallows socket errors without FSM cleanup                    |  exception routing              |
| Ad-hoc state variables instead of the 6-state FSM                                 | `fsm_specification.md`            |
