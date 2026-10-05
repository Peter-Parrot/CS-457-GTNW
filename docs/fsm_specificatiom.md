stateDiagram-v2
    [*] --> LOBBY : CONNECT (Player 1)

    state LOBBY {
        [*] --> Waiting_For_Player
        Waiting_For_Player --> Waiting_For_Player : LOBBY_WAIT
        Waiting_For_Player --> [*] : CONNECT (Player 2)
    }

    LOBBY --> IN_GAME : GAME_START

    state IN_GAME {
        Turn_Tracker --> Player_Input : Server sends YOUR_TURN & WAIT to respective clients
        Player_Input --> Move_Validation : Current turn client sends MOVE to server
        Player_Input --> Player_Input : Not current turn client waits
        Move_Validation --> Player_Input : Server sends ILLEGAL_MOVE / ERROR to current turn client
        Move_Validation --> Execute_Move : Valid Move
        Execute_Move --> Turn_Tracker : Win or draw condition not met, Server sends ALERT & STATE_UPDATE
        Execute_Move --> EndState : Win or draw condition met
        EndState --> [*] : Server sends STATE_UPDATE and final score TO clients
    }

    state PAUSED{
        Waiting_For_Reconnect --> Waiting_For_Reconnect : RECONNECT_WAIT
    }
    IN_GAME --> PAUSED : DROPPED_PLAYER
    IN_GAME --> LOBBY : DISCONNECT Player has quit the game
    PAUSED --> IN_GAME : CONNECT
    IN_GAME --> LOBBY: GAME_OVER, reset for new game