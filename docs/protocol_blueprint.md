### Framing Rules For Length-Prefixed Framing

Length-Prefixed Framing will be used for all messages sent to and from the server. Each message will have a 2 byte header that will specifiy the byte length of the message. The message payloads will be JSON objects as described below.






### Message Types

1. `CONNECT` (Client -> Server): Request to join the game room.
2. `LOBBY_WAIT` (Server -> Client): Notification that server is waiting for Player 2.
3. `GAME_START` (Server -> Clients): Game initiated, assigns roles (e.g. Player X vs Player O).
4. `YOUR_TURN` (Server -> Client): Inform player that it is their turn.
5. `MOVE` (Client -> Server): Player action (e.g., cell coordinates or answer choice).
6. `WAIT` (Server -> Client): Player needs to wait fo the other player to take their turn.
7. `ILLEGAL_MOVE` (Server -> Client): Player has made an illegal move, player needs to re-do their move.n a vague alert
8. `ALERT` (Server -> Clients): Inform players of game conditions (Missle launches, radiation levels, DEFCON levels, etc.).
9. `STATE_UPDATE` (Server -> Clients): Broadcast current game board / state and active player turn.
10. `GAME_OVER` (Server -> Clients): Victory / Draw notification with final scores.
11. `ERROR` (Server -> Client): Invalid move or malformed packet error.
12. `DISCONNECT` (Client -> Server): Client notifies server of intentional departure/quit.
13. `DROPPED_PLAYER` (Server -> Client): A player's connection has been lost
14. `RECONNECT_WAIT` (Server -> Client): Waiting for the dropped player to reconnect

1. CONNECT (Client -> Server)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        session_id (String): A unique identifier for the client.

    Rules: Initiates the connection. If the session_id is new, the server places the player in the lobby. If it matches a previously dropped player, the server bypasses the lobby and injects the new socket directly back into the paused player slot.

2. LOBBY_WAIT (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        message (String): Standardized text (e.g., "Waiting for opponent...").

    Rules: Sent when Player 1 connects while the server is still blocking on accept() waiting for Player 2's socket.

3. GAME_START (Server -> Clients)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        role (String): Assigns the player identifier (e.g., "Player 1", "Player 2").

    Rules: Broadcast simultaneously to both clients to break the lobby loop and initialize the 10x10 hidden grids.

4. YOUR_TURN (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.

    Rules: Sent exclusively to the active player. This control flag acts as a UI lock, triggering Python's input() function on the client to unfreeze their terminal and allow keyboard typing.

5. MOVE (Client -> Server)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        action (String): The desired command.
        coordinate (String): The targeted cell (e.g., "D4").

    Rules: The action field is strictly limited to "scan", "build abm", or "launch". This payload is treated as a proposed request, not a finalized fact. (More actions will be added down the road)

6. WAIT (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.

    Rules: Sent simultaneously with YOUR_TURN to the opposing spectator player. It forces their terminal to loop back to a listening state, preventing them from typing or interacting while the server blocks for the active player.

7. ILLEGAL_MOVE (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        reason (String): Explanation of the rule violation (e.g., insufficient resource budget).

    Rules: Triggered if a MOVE payload fails validation. The active player's turn does not end, and the sctive player must choose a valid move.

8. ALERT (Server -> Clients)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        message (String): Narrative or tactical updates.

    Rules: Broadcasts high-stakes game conditions without altering the turn flow. Used to push global warnings like "WARNING: LAUNCH DETECTED. IMPACT IN 2 TURNS." or vague recon data like "High radiation detected".

9. STATE_UPDATE (Server -> Clients)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        status (String): Current phase or active player.
        board (Array of Integers): The serialized grid payload.

    Rules: Pushed by the server after successfully resolving a legal move and updating the game logic. Forces both client terminals to redraw their visible board states.

10. GAME_OVER (Server -> Clients)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        result (String): "Win", "Loss", or "Draw".
        message (String): The cinematic conclusion text.

    Rules: Halts the central state machine. Triggered if the server detects a Decapitation Strike, an Asymmetric Survival win, or a Mutually Assured Destruction tie.

11. ERROR (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        error_message (String): Technical diagnostic string.

    Rules: Handles system-level exceptions like malformed JSON parsing or unsupported packet structures before the game logic attempts to evaluate them.

12. DISCONNECT (Client -> Server)

    Fields:
    
        timestamp (Number): Unix epoch timestamp of the message.

    Rules: Notifies the server of an intentional departure (e.g., the user types a quit command). The server can use this to instantly assign a forfeit victory rather than pausing the game to wait for a reconnect.

13. DROPPED_PLAYER (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        message (String): e.g., "Opponent disconnected. Waiting for them to return...".

    Rules: Triggered when the server's recv() function catches an empty byte payload, ConnectionResetError, or BrokenPipeError. The server pauses the main game loop, retains the board state in memory, and pushes this alert to the surviving player.

14. RECONNECT_WAIT (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.
        message (String): e.g., "Reconnected! Restoring game state...".

    Rules: Transmitted to the returning client immediately after they re-establish their TCP socket. It is followed instantly by a STATE_UPDATE payload to bring the dropped terminal back up to speed before resuming the turn loop.




```json
{
  "definitions": {
    "CONNECT": {
      "type": "object",
      "properties": {
        "type": { "const": "CONNECT" },
        "timestamp": { "type": "number" },
        "session_id": { "type": "string" }
      },
      "required": ["type", "timestamp", "session_id"]
    },
    "LOBBY_WAIT": {
      "type": "object",
      "properties": {
        "type": { "const": "LOBBY_WAIT" },
        "timestamp": { "type": "number" },
        "message": { "type": "string" }
      },
      "required": ["type", "timestamp", "message"]
    },
    "GAME_START": {
      "type": "object",
      "properties": {
        "type": { "const": "GAME_START" },
        "timestamp": { "type": "number" },
        "role": { "type": "string" }
      },
      "required": ["type", "timestamp", "role"]
    },
    "YOUR_TURN": {
      "type": "object",
      "properties": {
        "type": { "const": "YOUR_TURN" },
        "timestamp": { "type": "number" }
      },
      "required": ["type", "timestamp"]
    },
    "MOVE": {
      "type": "object",
      "properties": {
        "type": { "const": "MOVE" },
        "timestamp": { "type": "number" },
        "action": { "type": "string", "enum": ["scan", "build abm", "launch"] },
        "coordinate": { "type": "string" }
      },
      "required": ["type", "timestamp", "action", "coordinate"]
    },
    "WAIT": {
      "type": "object",
      "properties": {
        "type": { "const": "WAIT" },
        "timestamp": { "type": "number" }
      },
      "required": ["type", "timestamp"]
    },
    "ILLEGAL_MOVE": {
      "type": "object",
      "properties": {
        "type": { "const": "ILLEGAL_MOVE" },
        "timestamp": { "type": "number" },
        "reason": { "type": "string" }
      },
      "required": ["type", "timestamp", "reason"]
    },
    "ALERT": {
      "type": "object",
      "properties": {
        "type": { "const": "ALERT" },
        "timestamp": { "type": "number" },
        "message": { "type": "string" }
      },
      "required": ["type", "timestamp", "message"]
    },
    "STATE_UPDATE": {
      "type": "object",
      "properties": {
        "type": { "const": "STATE_UPDATE" },
        "timestamp": { "type": "number" },
        "status": { "type": "string" },
        "board": { "type": "array", "items": { "type": "integer" } }
      },
      "required": ["type", "timestamp", "status", "board"]
    },
    "GAME_OVER": {
      "type": "object",
      "properties": {
        "type": { "const": "GAME_OVER" },
        "timestamp": { "type": "number" },
        "result": { "type": "string" },
        "message": { "type": "string" }
      },
      "required": ["type", "timestamp", "result", "message"]
    },
    "ERROR": {
      "type": "object",
      "properties": {
        "type": { "const": "ERROR" },
        "timestamp": { "type": "number" },
        "error_message": { "type": "string" }
      },
      "required": ["type", "timestamp", "error_message"]
    },
    "DISCONNECT": {
      "type": "object",
      "properties": {
        "type": { "const": "DISCONNECT" },
        "timestamp": { "type": "number" }
      },
      "required": ["type", "timestamp"]
    },
    "DROPPED_PLAYER": {
      "type": "object",
      "properties": {
        "type": { "const": "DROPPED_PLAYER" },
        "timestamp": { "type": "number" },
        "message": { "type": "string" }
      },
      "required": ["type", "timestamp", "message"]
    },
    "RECONNECT_WAIT": {
      "type": "object",
      "properties": {
        "type": { "const": "RECONNECT_WAIT" },
        "timestamp": { "type": "number" },
        "message": { "type": "string" }
      },
      "required": ["type", "timestamp", "message"]
    }
  }
}
```
