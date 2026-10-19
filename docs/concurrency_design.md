# Non-Blocking I/O Multiplexing (selectors / select.select)


## Detailed explanation of select/selectors concurrency model .

* This game will use non-blocking multiplexing to implement concurrency and seperatinon between the game and network states. An event loop on the server will monitor the player connected sockets for when those sockets are ready to be serviced. Using the event loop means that the game will not be blocked by waiting for incoming player moves.

* The event loop will also simplify handling dropped conections from  players. Reading the sockets will be wrapped in a try/except block to catch when a players connection drops gracefully or abruptly. The event loop will catch errors and pause the game until the dropped player reconnects, the remaining player disconnects, or a timeout occurs.

* In the event of a player disconnecting the socket that the player was connected via will be closed and the event loop will listen for a new connection from that dropped player and create a new socket, and send the game state to the reconnected player.

### Listening socket accept
* The server creates a listening TCP socket that is configured to non-blocking mode (sock.setblocking(false)).
* The server's event loop calls selector.select() to check for client connections.
* When the server sees the TCP handshake(SYN) of a client it calls accept() and responds(SYN-ACK).

### Client registration
* The client is registered to a new non-blocking socket (sock.setblocking(false))
* The client socket is registered to a player within the game.


### Active game loop servicing
* The event loop listens for network packets on the player socket(s).
* The event loop takes player moves and validates them>
* Invalid moves cause a message to be sent to the active player making them redo their move.
* Valid moves are executed and the game board is updated accordingly.
* The clients are updated to the new state of the game board.
* Once the clients are updated on the state of the gameboard, the current active player switches.

### Client socket teardown
* Try/except block used to catch dropped connections and pause the game.
* When a player is disconnected the server unregisters that players socket and closes that connection.
* The server goes back to listening for a new connection waiting for the dropped player to reconect. 


# Thread safety analysis and lock acquisition strategy for event loop isolation.


## Shared resources
* Game Boards: The grid of the map representing the world map and the locations of startegic resources within the grid. Two matrices, one for each player that shows the respective players resources and applied fog of war over the resources of the other palyer that have not yet been discovered.(type=Matrices)
* Player Resources: The state of each players respective resources used to fund move actions.(type=list)
* Current Active Player: Which player is currently taking their turn.(type=integer)
* Client Sockets: The file descriptors for the connected player's sockets. (type=file descriptors)
* Score: The up to date state of the player's scores/status. (type=matrix)

## Single-threaded execution state safety
* Using multiplexing means that there will only be one thread running that will service one socket at a time, there is no threat of race conditions because there are no parallel threads. Locks will not be used because of this one at a time reading of sockets.

## Event-driven isolation
* Because it may be possible for the server to receive a move from the player who is waiting for their turn during the active player's turn, the event loop will check received moves against which player is currently the active player. Moves received from the inactive player will be ignored. 

* When an event is serviced in the single thread event loop, the event is completely executed before a new event is serviced, making it impossible for multiple events to try and change the games state at the same time. 






