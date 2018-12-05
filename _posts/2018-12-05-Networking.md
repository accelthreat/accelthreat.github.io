---
layout: default
title: TankWarz-Dev Diary 2 Networking
excerpt: TankWarz was the first networked game I ever worked upon and learning the ropes of multiplayer networking was a fun but very frustrating experience for me. This is the journey of how I designed the TankWarz network.
published: true
---

# Dev Diary 2: Networking

TankWarz was the first networked game I ever worked upon and learning the ropes of multiplayer networking was a fun but very frustrating experience for me. This is the journey of how I designed the TankWarz network.

## Finding A Library

After a lot of searching and reading articles and tutorials, one thing became clear. TankWarz was gonna need a UDP backend. TCP/IP was fine enough for LAN games but because the game was realtime and I wasn't gonna apply optimizations, I decided that going for UDP would be a safer bet.

After deciding on the UDP vs TCP debate, the next question was what library/framework should I use for the game. Time was a very limiting factor here as too low-level a library would consume too much of my time. (I was wrong though, in this assumption though, as I still became stuck for days on the framework I used). Python sockets was immediately out of the running as it was too low level. [Twisted](https://github.com/twisted/twisted) was a library I considered for a while, but I also decided against it as I read some articles that it was too complicated or too comprehensive. I also searched specifically for multiplayer libraries for Python such as [PodSixNet](https://github.com/chr15m/PodSixNet/) but this library was built on TCP so I ruled it out. Finally, I stumbled upon [Legume](https://github.com/katerd/legume) a python UDP multiplayer library which was what I needed.

## Architecture

I had a library in hand, but I didn't have any idea how to implement networking in the game. Fortunately, many others have already stumbled upon this problem and have written solutions. One such article I read was this, [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking) a guide explaining how Valve's networking architecture work. This, combined with [State Synchronization](https://gafferongames.com/post/state_synchronization/) by Gaffer on Games, provided me a general knowledge to implement these on TankWarz.

The basic idea is a Server-Client architecture where the server updates the clients about 60 times per second on the input and state of all the GameObjects in the simulation. The client sends input commands to the server and waits for a message from the server.

![server-client.png]({{site.baseurl}}/images/server-client.png)

### Server

The server is the real game itself and handles the clients input and state. Any conflict between the client and the server is quickly resolved because the server always overrides the client.

The server processes the client messages through a message handler:

``` python
def message_handler(self, sender, message):
    if legume.messages.message_factory.is_a(message, 'TankCreate'):
        self.logger.info("A server Tank has been created")
        if not self.time_running:
            self.start_timer()
        tanks[message.id.value] = SharedTank.create_from_message(message)
    elif legume.messages.message_factory.is_a(message, 'TankUpdate'):
        tanks[message.id.value].update_from_message(message)
    elif legume.messages.message_factory.is_a(message, 'TankCommand'):
        command = SharedTank.Command.from_int(message.command.value)
        tank = tanks[message.id.value]
    #More message types...
```

Every 1/60th of a second or based on the *UPDATE_RATE* the server sends updates to every client connected.

``` python
def _send_update(self):
    self.send_updates(self.server)
def go(self):
    self.server.listen(('', PORT))
    self.logger.info('Listening on port %d' % PORT)
    while True:
        if time.time() > self.update_timer + UPDATE_RATE:
            self._send_update()
            self.update_timer = time.time()
        self.server.update()
        time.sleep(0.0001)

def send_updates(self, server):
    self.lock.acquire()
    try:
        for tank in tanks.values():
            server.send_message_to_all(tank.get_message())
        for projectile in projectiles.values():
            server.send_message_to_all(projectile.get_message())
    finally:
        self.lock.release()
```

The method **send_updates** loops to every tank and projectile and sends updates to every client regarding its Input and State.

If a client connects for the first time, the server sends it an initial state with a map seed to allow the client to regenerate the map from scratch.

``` python
def join_request_handler(self, sender, args):
        self.send_initial_state(sender)
def send_initial_state(self, endpoint):
    self.logger.info("Connected to: %s" % endpoint)
    map_message = MapCreate()
    map_message.l.value = number_tile_x
    map_message.w.value = number_tile_y
    map_message.seed_a.value = int(self.seed_a)
    map_message.seed_b.value = int(self.seed_b)
    endpoint.send_message(map_message)
    self.lock.acquire()
    try:
        for tank in tanks.values():
            endpoint.send_message(tank.get_message())
        for projectile in projectiles.values():
            endpoint.send_message(projectile.get_message())
    finally:
        self.lock.release()
```

### Client

The client is an imitation or approximation of the real game running in the server. It has its own version of the game to perform client-side prediction when waiting for the real game state from the server.

As the client class is a big hot mess of local and networking code not abstracted away from each other. I'll only show the parts relevant to networking.

When the client first connects to the server, it sends a request to the server to create a new tank for it.

``` python
 def on_connect_accepted(self,sender, args):
    self.connected = True
    self.logger.info("Connection Accepted")
    if not self.started:
        self.logger.info("Creating New Tank")
        cl = ClientStart()
        cl.client_id.value = self.CLIENT_ID
        self._client.send_reliable_message(cl)
        message = TankCreate()
        message.id.value = self.cl_id
        #more message values...
        message.client_id.value = self.CLIENT_ID
        self._client.send_reliable_message(message)
        self.started = True
```

If a user inputs a command, the command is immediately applied to the client's tank to imitate a realtime response while sending the server a message and waiting for a response.

```python
def move_forward(self):
    self.tank.move(Direction.FORWARD)
    msg = TankCommand()
    msg.id.value = self.cl_id
    msg.command.value = Tank.Command.MOVE_FORWARD
    self.send_message_safely(msg)
### More tank command methods...
```

Here the client's tank is immediately moved forward. Then the client sends a message to the server to so that the server commands its version of the tank to move forward.

What follows is that the client receives the server's version of events in a message and applies it to its own tank. If the client and the server's tank state matches then the player does not notice anything. If it does not match, the client tank data will be overridden by the server's and a visible stuttering or jumping will occur.
