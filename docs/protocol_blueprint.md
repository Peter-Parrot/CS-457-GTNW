# Framing Rules For Length-Prefixed Framing

Length-Prefixed Framing will be used for all messages sent to and from the server. Each message will have a 2 byte header that will specifiy the byte length of the message. The message payloads will be JSON objects as described below.

### Byte Stream Examlpes

Example 1 (Turn Initialization): The server unlocks the active player's terminal and simultaneously pushes the updated board state.
```plaintext
[0x00, 0x32]{"type": "YOUR_TURN", "timestamp": 1728086779.123}[0x00, 0x69]{"type": "STATE_UPDATE", "timestamp": 1728086779.123, "status": "Player 1 Turn", "board": [0, 0, 1, 2]}
```

Example 2 (Player Actions): A client transmits multiple proposed moves in rapid succession (e.g., scanning a sector and building an interceptor).
```plaintext
[0x00, 0x53]{"type": "MOVE", "timestamp": 1728086779.123, "action": "scan", "coordinate": "B4"}[0x00, 0x58]{"type": "MOVE", "timestamp": 1728086779.123, "action": "build abm", "coordinate": "C4"}
```

Example 3 (Global Escalation): The server broadcasts a cinematic launch warning and immediately sends a control flag to lock the opposing player's terminal.
```plaintext
[0x00, 0x68]{"type": "ALERT", "timestamp": 1728086779.123, "message": "WARNING: LAUNCH DETECTED. IMPACT IN 2 TURNS."}[0x00, 0x2D]{"type": "WAIT", "timestamp": 1728086779.123}
```

Example 4 (Connection Recovery): The server notifies a surviving player of a drop, and then subsequently pushes a message when the connection is restored.
```plaintext
[0x00, 0x72]{"type": "DROPPED_PLAYER", "timestamp": 1728086779.123, "message": "Opponent disconnected. Waiting for them to return..."}[0x00, 0x64]{"type": "RECONNECT_WAIT", "timestamp": 1728086779.123, "message": "Reconnected! Restoring game state..."}
```

Example 5 (Mutually Assured Destruction): The state machine concludes the game, streaming a final status update followed by the cinematic conclusion text.
```plaintext
[0x00, 0x60]{"type": "STATE_UPDATE", "timestamp": 1728086779.123, "status": "Draw", "board": [0, 0, 1, 2]}[0x00, 0x84]{"type": "GAME_OVER", "timestamp": 1728086779.123, "result": "Draw", "message": "A strange game. The only winning move is not to play."}
```

# Message Extraction
*Read the message header to get the length of the message in bytes

*Read the number of bytes indicated in the header

*Convert the read message into a string

*Parse the string's message JSON object


# Connection Termination & Socket Lifecycle Management

## Graceful Disconnections (TCP FIN / 0-Byte EOF)
A graceful disconnection occurs when a player's client cleanly closes the connection with a DISCONNECT message.

When a payload containes b"", raise a ConnectionResetError to force the server to close the connection.


## Abrupt Terminations (TCP RST / Network Drops)
An abrupt termination happens when a player's internet drops, their computer loses power, or a TCP timeout occurs.

Catch ConnectionResetError, BrokenPipeError, and ConnectionAbortedError and wait for the player to reconnect to the game.

## The Pause and Recover Protocol
Whether a player drops gracefully or abruptly  the server must route the event into the same recovery block to protect the flow of the game.

    * Preserve the board: When a socket breaks, the server pauses the game to maintain state

    * Alert the remaining player: The server notifies the remaining player that the other player has disconnected and that it is waiting for them to return.

     * Block and wait: The server then blocks client input until the dropped player establishes reconnects.

    * Resume: Once the player reconnects, the server sends that player the current state of the game, game play is resumed.


# Message Types

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

#

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

15. RECONNECT_TIMEOUT (Server -> Client)

    Fields:

        timestamp (Number): Unix epoch timestamp of the message.

    Rules: Transmitted to the remaining client when the dropped client does not reconnect to the server after a predetermined amount of time. The state of the current game is dropped and the server resets for a new game. The remaining player is assigned a forfeit victory.

# JSON Schema

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
        "message": { "type": "string" }```
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
    },
    "RECONNECT_TIMEOUT": {
      "type": "object",
      "properties": {
        "type": { "const": "RECONNECT_TIMEOUT" },
        "timestamp": { "type": "number" },
        "message": { "type": "string" }
      },
      "required": ["type", "timestamp", "message"]
    }
  }
}
```

# Example Message Wire Streams

1. CONNECT (Byte Length: 85 | Hex: 0x0055)
```plaintext
[0x00, 0x55]{"type": "CONNECT", "timestamp": 1728086779.123, "session_id": "terminal_01"}
```
2. LOBBY_WAIT (Byte Length: 90 | Hex: 0x005A)
```plaintext
[0x00, 0x5A]{"type": "LOBBY_WAIT", "timestamp": 1728086779.123, "message": "Waiting for opponent..."}
```
3. GAME_START (Byte Length: 74 | Hex: 0x004A)
```plaintext
[0x00, 0x4A]{"type": "GAME_START", "timestamp": 1728086779.123, "role": "Player 1"}
```
4. YOUR_TURN (Byte Length: 50 | Hex: 0x0032)
```plaintext
[0x00, 0x32]{"type": "YOUR_TURN", "timestamp": 1728086779.123}
```
5. MOVE (Byte Length: 85 | Hex: 0x0055)
```plaintext
[0x00, 0x55]{"type": "MOVE", "timestamp": 1728086779.123, "action": "launch", "coordinate": "D4"}
```
6. WAIT (Byte Length: 45 | Hex: 0x002D)
```plaintext
[0x00, 0x2D]{"type": "WAIT", "timestamp": 1728086779.123}
```
7. ILLEGAL_MOVE (Byte Length: 95 | Hex: 0x005F)
```plaintext
[0x00, 0x5F]{"type": "ILLEGAL_MOVE", "timestamp": 1728086779.123, "reason": "Insufficient DEFCON budget."}
```
8. ALERT (Byte Length: 104 | Hex: 0x0068)
```plaintext
[0x00, 0x68]{"type": "ALERT", "timestamp": 1728086779.123, "message": "WARNING: LAUNCH DETECTED. IMPACT IN 2 TURNS."}
```
9. STATE_UPDATE (Byte Length: 119 | Hex: 0x0077)
```plaintext
[0x00, 0x77]{"type": "STATE_UPDATE", "timestamp": 1728086779.123, "status": "Player 2 Turn", "board": [0, 0, 1, 2, 0, 0, 0, 0, 0, 0]}
```
10. GAME_OVER (Byte Length: 132 | Hex: 0x0084)
```plaintext
[0x00, 0x84]{"type": "GAME_OVER", "timestamp": 1728086779.123, "result": "Draw", "message": "A strange game. The only winning move is not to play."}
```
11. ERROR (Byte Length: 97 | Hex: 0x0061)
```plaintext
[0x00, 0x61]{"type": "ERROR", "timestamp": 1728086779.123, "error_message": "Malformed JSON payload received."}
```
12. DISCONNECT (Byte Length: 51 | Hex: 0x0033)
```plaintext
[0x00, 0x33]{"type": "DISCONNECT", "timestamp": 1728086779.123}
```
13. DROPPED_PLAYER (Byte Length: 114 | Hex: 0x0072)
```plaintext
[0x00, 0x72]{"type": "DROPPED_PLAYER", "timestamp": 1728086779.123, "message": "Opponent disconnected. Waiting for them to return..."}
```
14. RECONNECT_WAIT (Byte Length: 100 | Hex: 0x0064)
```plaintext
[0x00, 0x64]{"type": "RECONNECT_WAIT", "timestamp": 1728086779.123, "message": "Reconnected! Restoring game state..."}
```
15. RECONNECT_TIMEOUT (Byte Length: 122 | Hex: 0x007A)
```plaintext
[0x00, 0x7A]{"type": "RECONNECT_TIMEOUT", "timestamp": 1728086779.123, "message": "Opponent failed to reconnect within 60 seconds. Match forfeited."}
```