# Protocol Message Schemas

This document defines the JSON schemas for the messages defined in the protocol specification.

## Schemas

```json
{
  "definitions": {
    "CONNECT": {
      "type": "object",
      "properties": {
        "type": { "const": "CONNECT" },
        "session_id": { "type": "string" }
      },
      "required": ["type", "session_id"]
    },
    "LOBBY_WAIT": {
      "type": "object",
      "properties": {
        "type": { "const": "LOBBY_WAIT" },
        "message": { "type": "string" }
      },
      "required": ["type", "message"]
    },
    "GAME_START": {
      "type": "object",
      "properties": {
        "type": { "const": "GAME_START" },
        "role": { "type": "string" }
      },
      "required": ["type", "role"]
    },
    "YOUR_TURN": {
      "type": "object",
      "properties": {
        "type": { "const": "YOUR_TURN" }
      },
      "required": ["type"]
    },
    "MOVE": {
      "type": "object",
      "properties": {
        "type": { "const": "MOVE" },
        "action": { "type": "string", "enum": ["scan", "build abm", "launch"] },
        "coordinate": { "type": "string" }
      },
      "required": ["type", "action", "coordinate"]
    },
    "WAIT": {
      "type": "object",
      "properties": {
        "type": { "const": "WAIT" }
      },
      "required": ["type"]
    },
    "ILLEGAL_MOVE": {
      "type": "object",
      "properties": {
        "type": { "const": "ILLEGAL_MOVE" },
        "reason": { "type": "string" }
      },
      "required": ["type", "reason"]
    },
    "ALERT": {
      "type": "object",
      "properties": {
        "type": { "const": "ALERT" },
        "message": { "type": "string" }
      },
      "required": ["type", "message"]
    },
    "STATE_UPDATE": {
      "type": "object",
      "properties": {
        "type": { "const": "STATE_UPDATE" },
        "status": { "type": "string" },
        "board": { "type": "array", "items": { "type": "integer" } }
      },
      "required": ["type", "status", "board"]
    },
    "GAME_OVER": {
      "type": "object",
      "properties": {
        "type": { "const": "GAME_OVER" },
        "result": { "type": "string" },
        "message": { "type": "string" }
      },
      "required": ["type", "result", "message"]
    },
    "ERROR": {
      "type": "object",
      "properties": {
        "type": { "const": "ERROR" },
        "error_message": { "type": "string" }
      },
      "required": ["type", "error_message"]
    },
    "DISCONNECT": {
      "type": "object",
      "properties": {
        "type": { "const": "DISCONNECT" }
      },
      "required": ["type"]
    },
    "DROPPED_PLAYER": {
      "type": "object",
      "properties": {
        "type": { "const": "DROPPED_PLAYER" },
        "message": { "type": "string" }
      },
      "required": ["type", "message"]
    },
    "RECONNECT_WAIT": {
      "type": "object",
      "properties": {
        "type": { "const": "RECONNECT_WAIT" },
        "message": { "type": "string" }
      },
      "required": ["type", "message"]
    }
  }
}
```
