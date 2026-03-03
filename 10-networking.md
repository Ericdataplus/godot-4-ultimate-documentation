# 10 — Networking & Multiplayer

> ENet, WebRTC, RPCs, synchronization, and multiplayer architecture.

---

## Table of Contents

- [Networking Overview](#networking-overview)
- [ENet Setup (Client-Server)](#enet-setup)
- [RPCs (Remote Procedure Calls)](#rpcs)
- [Spawning & Synchronization](#spawning--synchronization)
- [WebRTC (Peer-to-Peer)](#webrtc)
- [Lobby System](#lobby-system)
- [Multiplayer Best Practices](#best-practices)

---

## Networking Overview

```
Client-Server (ENet):
├── One player hosts, others connect
├── Server is authoritative (cheat prevention)
├── Requires port forwarding or relay server
└── Best for: Competitive games, MMOs

Peer-to-Peer (WebRTC):
├── All players connect directly to each other
├── No dedicated server needed
├── Built-in NAT traversal (mostly works through firewalls)
├── Requires signaling server for initial connection
└── Best for: Co-op games, small player counts

Steam Networking:
├── Uses GodotSteam plugin
├── Replaces ENet peer with Steam networking
├── NAT traversal handled by Steam
└── Best for: Steam-published games
```

---

## ENet Setup

### Server

```gdscript
# server.gd (on the host)
extends Node

const PORT: int = 7000
const MAX_CLIENTS: int = 4

var peer := ENetMultiplayerPeer.new()

func host_game() -> void:
    var error := peer.create_server(PORT, MAX_CLIENTS)
    if error != OK:
        push_error("Failed to create server: %s" % error)
        return
    
    multiplayer.multiplayer_peer = peer
    
    # Connect signals
    multiplayer.peer_connected.connect(_on_peer_connected)
    multiplayer.peer_disconnected.connect(_on_peer_disconnected)
    
    print("Server started on port %d" % PORT)

func _on_peer_connected(id: int) -> void:
    print("Player connected: %d" % id)

func _on_peer_disconnected(id: int) -> void:
    print("Player disconnected: %d" % id)
```

### Client

```gdscript
# client.gd
extends Node

func join_game(address: String, port: int = 7000) -> void:
    var peer := ENetMultiplayerPeer.new()
    var error := peer.create_client(address, port)
    if error != OK:
        push_error("Failed to connect: %s" % error)
        return
    
    multiplayer.multiplayer_peer = peer
    
    multiplayer.connected_to_server.connect(_on_connected)
    multiplayer.connection_failed.connect(_on_connection_failed)
    multiplayer.server_disconnected.connect(_on_server_disconnected)

func _on_connected() -> void:
    print("Connected to server! My ID: %d" % multiplayer.get_unique_id())

func _on_connection_failed() -> void:
    print("Connection failed!")

func _on_server_disconnected() -> void:
    print("Server disconnected!")
```

### Unified Network Manager

```gdscript
# network_manager.gd (Autoload)
extends Node

signal player_connected(id: int)
signal player_disconnected(id: int)
signal server_started
signal connection_established
signal connection_failed

const PORT: int = 7000
const MAX_CLIENTS: int = 8

func host() -> void:
    var peer := ENetMultiplayerPeer.new()
    peer.create_server(PORT, MAX_CLIENTS)
    multiplayer.multiplayer_peer = peer
    
    multiplayer.peer_connected.connect(func(id):
        player_connected.emit(id)
    )
    multiplayer.peer_disconnected.connect(func(id):
        player_disconnected.emit(id)
    )
    server_started.emit()

func join(address: String) -> void:
    var peer := ENetMultiplayerPeer.new()
    peer.create_client(address, PORT)
    multiplayer.multiplayer_peer = peer
    
    multiplayer.connected_to_server.connect(func():
        connection_established.emit()
    )
    multiplayer.connection_failed.connect(func():
        connection_failed.emit()
    )

func disconnect_from_game() -> void:
    multiplayer.multiplayer_peer = null

func is_server() -> bool:
    return multiplayer.is_server()

func get_my_id() -> int:
    return multiplayer.get_unique_id()
```

---

## RPCs

### @rpc Annotation

```gdscript
# The new Godot 4 RPC system:

# Any peer can call this, runs on everyone
@rpc("any_peer", "call_local", "reliable")
func chat_message(sender: String, message: String) -> void:
    chat_log.add_message(sender, message)

# Only the server/authority can call this
@rpc("authority", "call_local", "reliable")
func spawn_enemy(pos: Vector2) -> void:
    var enemy := enemy_scene.instantiate()
    enemy.global_position = pos
    add_child(enemy)

# Unreliable (for frequent updates like position)
@rpc("any_peer", "call_remote", "unreliable_ordered")
func sync_position(pos: Vector2) -> void:
    global_position = pos
```

### RPC Modes Explained

```
WHO CAN CALL:
"authority"   → Only the multiplayer authority (usually server)
"any_peer"    → Any connected peer can call this

WHERE IT RUNS:
"call_local"  → Runs on caller AND remote peers
"call_remote" → Runs ONLY on remote peers (not the caller)

RELIABILITY:
"reliable"           → Guaranteed delivery, ordered (TCP-like)
"unreliable"         → May drop packets, fastest (UDP-like)
"unreliable_ordered" → May drop but maintains order

CHANNEL (optional):
Channel number for parallel ordering
```

### Calling RPCs

```gdscript
# Call on ALL peers
chat_message.rpc("Player1", "Hello!")

# Call on SPECIFIC peer
chat_message.rpc_id(target_peer_id, "Server", "Welcome!")

# Call on SERVER only (peer ID 1)
request_spawn.rpc_id(1, position)
```

---

## Spawning & Synchronization

### MultiplayerSpawner

Automatically replicates node creation across peers:

```
1. Add MultiplayerSpawner to your scene
2. Set "Spawn Path" to the parent node where objects spawn
3. Add scenes to "Auto Spawn List"
4. Objects spawned under the spawn path are replicated
```

```gdscript
# Server spawns — clients automatically get copies
@onready var spawner: MultiplayerSpawner = $MultiplayerSpawner

func _ready() -> void:
    if multiplayer.is_server():
        spawner.spawn_function = custom_spawn

func custom_spawn(data: Variant) -> Node:
    var player := player_scene.instantiate()
    player.name = str(data)  # Use peer ID as name
    return player
```

### MultiplayerSynchronizer

Automatically syncs properties:

```
1. Add MultiplayerSynchronizer as child of the node to sync
2. Click "Replication" at bottom of editor
3. Add properties to sync (position, rotation, health, etc.)
4. Set sync mode: Always, On Change, or Never
```

```gdscript
# The authority (owner) of a node determines who sends updates
# Server-authoritative: server owns all nodes
# Client-authoritative: each client owns their player

# Set authority
node.set_multiplayer_authority(peer_id)

# Check if WE are the authority
if is_multiplayer_authority():
    # We control this node
    _process_input()
```

### Manual Sync (For Complex Cases)

```gdscript
# player.gd
extends CharacterBody2D

func _physics_process(delta: float) -> void:
    if is_multiplayer_authority():
        # Local player: handle input
        var direction := Input.get_vector("left", "right", "up", "down")
        velocity = direction * speed
        move_and_slide()
        
        # Send position to others (unreliable for speed)
        sync_state.rpc(global_position, velocity)

@rpc("any_peer", "call_remote", "unreliable_ordered")
func sync_state(pos: Vector2, vel: Vector2) -> void:
    # Remote players: interpolate to received position
    global_position = global_position.lerp(pos, 0.5)
    velocity = vel
```

---

## WebRTC

### Setting Up WebRTC P2P

```gdscript
# Requires a signaling server for initial connection
# (Can be a simple WebSocket server)

var rtc_peer := WebRTCMultiplayerPeer.new()

func create_mesh_peer(my_id: int) -> void:
    rtc_peer.create_mesh(my_id)
    multiplayer.multiplayer_peer = rtc_peer

func add_peer_connection(peer_id: int) -> void:
    var connection := WebRTCPeerConnection.new()
    connection.initialize({
        "iceServers": [
            {"urls": ["stun:stun.l.google.com:19302"]}
        ]
    })
    rtc_peer.add_peer(connection, peer_id)
    
    # Handle ICE candidates and SDP through signaling server
```

---

## Lobby System

```gdscript
# lobby_manager.gd (Autoload)
extends Node

signal lobby_updated(players: Dictionary)

var players: Dictionary = {}  # peer_id: {name, ready, color}

func _ready() -> void:
    multiplayer.peer_connected.connect(_on_player_connected)
    multiplayer.peer_disconnected.connect(_on_player_disconnected)

func _on_player_connected(id: int) -> void:
    # Request their info
    request_player_info.rpc_id(id)

func _on_player_disconnected(id: int) -> void:
    players.erase(id)
    lobby_updated.emit(players)

@rpc("any_peer", "reliable")
func request_player_info() -> void:
    var sender := multiplayer.get_remote_sender_id()
    # Send our info back
    receive_player_info.rpc_id(sender, {
        "name": Settings.player_name,
        "ready": false,
        "color": Settings.player_color,
    })

@rpc("any_peer", "reliable")
func receive_player_info(info: Dictionary) -> void:
    var sender := multiplayer.get_remote_sender_id()
    players[sender] = info
    lobby_updated.emit(players)

@rpc("any_peer", "call_local", "reliable")
func set_ready(ready: bool) -> void:
    var sender := multiplayer.get_remote_sender_id()
    if players.has(sender):
        players[sender]["ready"] = ready
        lobby_updated.emit(players)

# Server checks if all ready
func check_all_ready() -> bool:
    return players.values().all(func(p): return p["ready"])
```

---

## Best Practices

```
1. SERVER AUTHORITY: Never trust the client.
   The server validates all gameplay actions.

2. INTERPOLATION: Smooth out network jitter with
   position interpolation, not raw position sets.

3. CLIENT PREDICTION: Move the local player immediately,
   correct with server state later.

4. DELTA COMPRESSION: Only send what changed,
   not the entire state every frame.

5. UNRELIABLE FOR POSITION: Use unreliable_ordered for
   frequently-updated data (position, rotation).
   Use reliable for important events (damage, death, chat).

6. UNIQUE IDS: Use multiplayer.get_unique_id() for
   player identification, never hardcode IDs.

7. AUTHORITY CHECK: Always check is_multiplayer_authority()
   before processing input/gameplay logic.

8. DISCONNECT HANDLING: Always handle disconnections
   gracefully — clean up player nodes and state.
```

---

*← [09 — Navigation & AI](./09-navigation-and-ai.md) | [11 — Save & Load](./11-save-and-load.md) →*
