## Non-Blocking I/O Multiplexing (selectors / select.select)


### Detailed explanation of select/selectors concurrency model .

* This game will use non-blocking multiplexing to implement concurrency and seperatinon between the game and network states. An event loop on the server will monitor the player connected sockets for when those sockets are ready to be serviced. Using the event loop means that the game will not be blocked by waiting for incoming player moves.

* The event loop will also simplify handling dropped conections from  players. Reading the sockets will be wrapped in a try/except block to catch when a players connection drops gracefully or abruptly. The event loop will catch errors and pause the game until the dropped player reconnects, the remaining player disconnects, or a timeout occurs.

* In the event of a player disconnecting the socket that the player was connected via will be closed and the event loop will listen for a new connection from that dropped player and create a new socket, and send the game state to the reconnected player.

### Thread safety analysis and lock acquisition strategy for event loop isolation.

* Using multiplexing means that there will only be one thread running that will service one socket at a time, there is no threat of race conditions because there are no parallel threads. Locks will not be used because of this one at a time reading of sockets.

* Because it may be possible for the server to receive a move from the player who is waiting for their turn during the active player's turn, the event loop will check received moves against which player is currently the active player. Moves received from the inactive player will be ignored. 






