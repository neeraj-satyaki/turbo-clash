# Turbo Clash MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a playable top-down 2D multiplayer combat racing game with LAN support for up to 10 players.

**Architecture:** Godot 4 client with GDScript handles all game logic, rendering, physics, and ENet networking. Host-authoritative model where one player runs the simulation and peers send inputs. No backend server needed for MVP.

**Tech Stack:** Godot 4.3+, GDScript, ENet (built-in), Godot physics 2D

---

## File Structure

```
turbo-clash/
├── project.godot                    # Godot project config
├── export_presets.cfg               # Export configs (desktop)
├── scripts/
│   ├── autoload/
│   │   ├── game_manager.gd          # Game state machine (lobby→race→results)
│   │   └── network_manager.gd       # ENet host/join, player registry, host migration
│   ├── car/
│   │   ├── car_body.gd              # Car physics, movement, input handling
│   │   ├── car_stats.gd             # Speed, acceleration, handling resource
│   │   ├── car_visual.gd            # Sprite, color, rotation visual sync
│   │   └── car_stunt.gd             # Drift detection, barrel roll, boost meter
│   ├── weapons/
│   │   ├── weapon_base.gd           # Base weapon class
│   │   ├── missile.gd               # Forward-firing homing missile
│   │   ├── oil_slick.gd             # Drop behind, causes spin-out
│   │   └── shield.gd                # Blocks one hit
│   ├── track/
│   │   ├── track_data.gd            # Track resource: waypoints, boundaries, spawns
│   │   ├── checkpoint.gd            # Checkpoint area, lap counting
│   │   ├── weapon_pickup.gd         # Weapon spawn point, respawn timer
│   │   └── ramp.gd                  # Ramp launcher for stunts
│   ├── ui/
│   │   ├── main_menu.gd             # Main menu screen
│   │   ├── lobby_screen.gd          # Pre-race lobby, player list, countdown
│   │   ├── hud.gd                   # In-race HUD: position, lap, HP, boost, weapon
│   │   └── results_screen.gd        # Post-race results, XP summary
│   └── network/
│       ├── player_input.gd          # Input capture and serialization
│       ├── sync_manager.gd          # State broadcast (60/s), event relay
│       └── host_authority.gd        # Host-side input validation, state reconciliation
├── scenes/
│   ├── car/
│   │   └── car.tscn                 # Car scene (CharacterBody2D + sprites + collision)
│   ├── weapons/
│   │   ├── missile.tscn
│   │   ├── oil_slick.tscn
│   │   └── shield.tscn
│   ├── track/
│   │   ├── track_oval.tscn          # Simple oval track
│   │   ├── track_ramps.tscn         # Track with ramps and obstacles
│   │   ├── checkpoint.tscn
│   │   ├── weapon_pickup.tscn
│   │   └── ramp.tscn
│   ├── ui/
│   │   ├── main_menu.tscn
│   │   ├── lobby_screen.tscn
│   │   ├── hud.tscn
│   │   └── results_screen.tscn
│   └── main.tscn                    # Entry scene, loads main menu
├── resources/
│   ├── cars/
│   │   ├── car_default.tres         # Default car stats resource
│   │   └── car_colors.tres          # 3 color variations
│   └── tracks/
│       ├── track_oval_data.tres      # Oval track waypoints/boundaries
│       └── track_ramps_data.tres     # Ramps track data
├── assets/
│   ├── sprites/
│   │   ├── car_body.png
│   │   ├── car_wheel.png
│   │   ├── missile.png
│   │   ├── oil_slick.png
│   │   ├── shield_bubble.png
│   │   ├── weapon_pickup_box.png
│   │   ├── ramp.png
│   │   ├── track_tile_road.png
│   │   ├── track_tile_grass.png
│   │   ├── track_tile_barrier.png
│   │   └── boost_particle.png
│   ├── ui/
│   │   ├── hp_heart.png
│   │   ├── boost_bar_fill.png
│   │   ├── weapon_slot_bg.png
│   │   └── position_badge.png
│   └── audio/
│       ├── engine_loop.wav
│       ├── missile_fire.wav
│       ├── oil_splat.wav
│       ├── shield_activate.wav
│       ├── hit_impact.wav
│       ├── drift_screech.wav
│       ├── boost_whoosh.wav
│       ├── countdown_beep.wav
│       └── race_finish.wav
└── tests/
    ├── test_car_physics.gd
    ├── test_weapon_logic.gd
    ├── test_stunt_system.gd
    ├── test_checkpoint.gd
    └── test_network_sync.gd
```

---

### Task 1: Godot Project Scaffold

**Files:**
- Create: `project.godot`
- Create: `scenes/main.tscn`
- Create: `scripts/autoload/game_manager.gd`

- [ ] **Step 1: Create Godot project**

Create the project file. In your terminal:

```bash
cd /media/subhadra/A0822F77822F50D8/Projects/turbo-clash
mkdir -p scripts/autoload scripts/car scripts/weapons scripts/track scripts/ui scripts/network
mkdir -p scenes/car scenes/weapons scenes/track scenes/ui
mkdir -p resources/cars resources/tracks
mkdir -p assets/sprites assets/ui assets/audio
mkdir -p tests
```

- [ ] **Step 2: Create project.godot**

```ini
; Engine configuration file.
; It's best edited using the editor UI and not directly,
; but it can also be edited by hand.

config_version=5

[application]

config/name="Turbo Clash"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.3", "GL Compatibility")

[autoload]

GameManager="*res://scripts/autoload/game_manager.gd"
NetworkManager="*res://scripts/autoload/network_manager.gd"

[display]

window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"
window/stretch/aspect="expand"

[input]

accelerate={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":87,"key_label":0,"unicode":119)]
}
brake={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":83,"key_label":0,"unicode":115)]
}
steer_left={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":65,"key_label":0,"unicode":97)]
}
steer_right={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":68,"key_label":0,"unicode":100)]
}
fire_weapon={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":32,"key_label":0,"unicode":32)]
}
activate_boost={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194325,"key_label":0,"unicode":0)]
}

[layer_names]

2d_physics/layer_1="cars"
2d_physics/layer_2="track_boundaries"
2d_physics/layer_3="checkpoints"
2d_physics/layer_4="weapon_pickups"
2d_physics/layer_5="projectiles"
2d_physics/layer_6="ramps"

[physics]

2d/default_gravity=0
```

- [ ] **Step 3: Create game_manager.gd (state machine)**

```gdscript
# scripts/autoload/game_manager.gd
extends Node

enum GameState { MENU, LOBBY, COUNTDOWN, RACING, RESULTS }

signal state_changed(old_state: GameState, new_state: GameState)

var current_state: GameState = GameState.MENU
var players: Dictionary = {}  # peer_id -> { name, color_index, car_node, position, lap, hp }
var race_config: Dictionary = {
	"track": "oval",
	"laps": 3,
	"max_players": 10,
}

func change_state(new_state: GameState) -> void:
	var old_state = current_state
	current_state = new_state
	state_changed.emit(old_state, new_state)

func register_player(peer_id: int, player_name: String, color_index: int) -> void:
	players[peer_id] = {
		"name": player_name,
		"color_index": color_index,
		"car_node": null,
		"position": 0,
		"lap": 0,
		"hp": 3,
		"boost_meter": 0.0,
		"weapon": "",
		"finished": false,
		"finish_time": 0.0,
	}

func remove_player(peer_id: int) -> void:
	if players.has(peer_id):
		players.erase(peer_id)

func reset_race() -> void:
	for peer_id in players:
		players[peer_id]["lap"] = 0
		players[peer_id]["hp"] = 3
		players[peer_id]["boost_meter"] = 0.0
		players[peer_id]["weapon"] = ""
		players[peer_id]["finished"] = false
		players[peer_id]["finish_time"] = 0.0

func get_rankings() -> Array:
	var ranked = players.values().duplicate()
	ranked.sort_custom(func(a, b):
		if a["finished"] and not b["finished"]:
			return true
		if not a["finished"] and b["finished"]:
			return false
		if a["finished"] and b["finished"]:
			return a["finish_time"] < b["finish_time"]
		if a["lap"] != b["lap"]:
			return a["lap"] > b["lap"]
		return a["position"] > b["position"]
	)
	return ranked
```

- [ ] **Step 4: Create empty main scene**

Create `scenes/main.tscn`:

```
[gd_scene format=3 uid="uid://main_scene"]

[node name="Main" type="Node2D"]
```

- [ ] **Step 5: Verify project opens in Godot**

```bash
# Open in Godot editor to verify no errors
# If running headless/CI, validate with:
godot --headless --quit 2>&1 | head -20
```

Expected: No errors. Project loads with main scene.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: scaffold Godot project with game manager autoload"
```

---

### Task 2: Network Manager (ENet LAN)

**Files:**
- Create: `scripts/autoload/network_manager.gd`

- [ ] **Step 1: Write network_manager.gd**

```gdscript
# scripts/autoload/network_manager.gd
extends Node

signal player_connected(peer_id: int)
signal player_disconnected(peer_id: int)
signal connection_succeeded()
signal connection_failed()
signal server_created()

const DEFAULT_PORT := 7777
const MAX_PLAYERS := 10

var peer: ENetMultiplayerPeer = null
var is_host := false
var local_player_name := "Player"
var local_color_index := 0

func create_server(port: int = DEFAULT_PORT) -> Error:
	peer = ENetMultiplayerPeer.new()
	var error = peer.create_server(port, MAX_PLAYERS)
	if error != OK:
		return error
	multiplayer.multiplayer_peer = peer
	is_host = true
	_connect_signals()
	GameManager.register_player(1, local_player_name, local_color_index)
	server_created.emit()
	return OK

func join_server(address: String, port: int = DEFAULT_PORT) -> Error:
	peer = ENetMultiplayerPeer.new()
	var error = peer.create_client(address, port)
	if error != OK:
		return error
	multiplayer.multiplayer_peer = peer
	is_host = false
	_connect_signals()
	return OK

func disconnect_from_game() -> void:
	if peer:
		peer.close()
		peer = null
	multiplayer.multiplayer_peer = null
	is_host = false
	GameManager.players.clear()

func _connect_signals() -> void:
	multiplayer.peer_connected.connect(_on_peer_connected)
	multiplayer.peer_disconnected.connect(_on_peer_disconnected)
	multiplayer.connected_to_server.connect(_on_connected_to_server)
	multiplayer.connection_failed.connect(_on_connection_failed)

func _on_peer_connected(peer_id: int) -> void:
	player_connected.emit(peer_id)

func _on_peer_disconnected(peer_id: int) -> void:
	GameManager.remove_player(peer_id)
	player_disconnected.emit(peer_id)

func _on_connected_to_server() -> void:
	var my_id = multiplayer.get_unique_id()
	rpc_id(1, "register_peer", local_player_name, local_color_index)
	connection_succeeded.emit()

func _on_connection_failed() -> void:
	disconnect_from_game()
	connection_failed.emit()

@rpc("any_peer", "reliable")
func register_peer(player_name: String, color_index: int) -> void:
	var sender_id = multiplayer.get_remote_sender_id()
	GameManager.register_player(sender_id, player_name, color_index)
	# Send existing player list to new player
	for existing_id in GameManager.players:
		if existing_id != sender_id:
			var p = GameManager.players[existing_id]
			rpc_id(sender_id, "add_existing_player", existing_id, p["name"], p["color_index"])
	# Notify all others about new player
	for existing_id in GameManager.players:
		if existing_id != sender_id and existing_id != 1:
			rpc_id(existing_id, "add_existing_player", sender_id, player_name, color_index)

@rpc("authority", "reliable")
func add_existing_player(peer_id: int, player_name: String, color_index: int) -> void:
	GameManager.register_player(peer_id, player_name, color_index)

func broadcast_lan_discovery() -> void:
	var udp = PacketPeerUDP.new()
	udp.set_broadcast_enabled(true)
	udp.set_dest_address("255.255.255.255", DEFAULT_PORT + 1)
	var data = ("TURBO_CLASH:" + local_player_name).to_utf8_buffer()
	udp.put_packet(data)
	udp.close()

func listen_lan_discovery(callback: Callable) -> PacketPeerUDP:
	var udp = PacketPeerUDP.new()
	udp.bind(DEFAULT_PORT + 1)
	# Caller polls udp.get_available_packet_count() in _process
	return udp
```

- [ ] **Step 2: Test locally — create and join**

Open two Godot instances (or use the editor + exported build). One creates server, other joins via `127.0.0.1`. Verify `player_connected` signal fires on both.

- [ ] **Step 3: Commit**

```bash
git add scripts/autoload/network_manager.gd
git commit -m "feat: add ENet network manager with LAN host/join and player registry"
```

---

### Task 3: Car Physics (Single Player)

**Files:**
- Create: `scripts/car/car_body.gd`
- Create: `scripts/car/car_stats.gd`
- Create: `scripts/car/car_visual.gd`
- Create: `scenes/car/car.tscn`

- [ ] **Step 1: Create car_stats.gd resource**

```gdscript
# scripts/car/car_stats.gd
extends Resource
class_name CarStats

@export var max_speed := 500.0         # pixels/sec
@export var acceleration := 800.0      # pixels/sec^2
@export var brake_force := 1200.0      # pixels/sec^2
@export var friction := 400.0          # passive deceleration
@export var steer_speed := 3.0         # radians/sec
@export var drift_steer_mult := 1.4    # steering multiplier while drifting
@export var drift_friction := 200.0    # reduced friction while drifting
```

- [ ] **Step 2: Create car_visual.gd**

```gdscript
# scripts/car/car_visual.gd
extends Sprite2D
class_name CarVisual

const COLORS: Array[Color] = [
	Color(0.9, 0.2, 0.2),  # Red
	Color(0.2, 0.5, 0.9),  # Blue
	Color(0.2, 0.9, 0.3),  # Green
]

var color_index := 0

func set_car_color(index: int) -> void:
	color_index = clampi(index, 0, COLORS.size() - 1)
	modulate = COLORS[color_index]
```

- [ ] **Step 3: Create car_body.gd**

```gdscript
# scripts/car/car_body.gd
extends CharacterBody2D
class_name CarBody

@export var stats: CarStats

var speed := 0.0
var steer_input := 0.0
var is_accelerating := false
var is_braking := false
var is_drifting := false
var is_local_player := false
var peer_id := 0
var hp := 3
var current_weapon := ""
var is_spinning_out := false
var spinout_timer := 0.0

const SPINOUT_DURATION := 1.5
const SPINOUT_SLOW_MULT := 0.3

func _ready() -> void:
	if stats == null:
		stats = CarStats.new()

func _physics_process(delta: float) -> void:
	if is_spinning_out:
		_process_spinout(delta)
		return

	if is_local_player:
		_read_input()

	_apply_steering(delta)
	_apply_acceleration(delta)
	_apply_friction(delta)

	velocity = transform.x * speed
	move_and_slide()

func _read_input() -> void:
	is_accelerating = Input.is_action_pressed("accelerate")
	is_braking = Input.is_action_pressed("brake")
	steer_input = Input.get_axis("steer_left", "steer_right")

func _apply_steering(delta: float) -> void:
	if abs(speed) < 10.0:
		return
	var steer_amount = stats.steer_speed
	if is_drifting:
		steer_amount *= stats.drift_steer_mult
	rotation += steer_input * steer_amount * delta * sign(speed)

func _apply_acceleration(delta: float) -> void:
	if is_accelerating:
		speed = move_toward(speed, stats.max_speed, stats.acceleration * delta)
	elif is_braking:
		speed = move_toward(speed, -stats.max_speed * 0.3, stats.brake_force * delta)

func _apply_friction(delta: float) -> void:
	var friction_amount = stats.friction
	if is_drifting:
		friction_amount = stats.drift_friction
	if not is_accelerating and not is_braking:
		speed = move_toward(speed, 0.0, friction_amount * delta)

func apply_spinout() -> void:
	is_spinning_out = true
	spinout_timer = SPINOUT_DURATION
	speed *= SPINOUT_SLOW_MULT

func _process_spinout(delta: float) -> void:
	spinout_timer -= delta
	rotation += 10.0 * delta  # visual spin
	speed = move_toward(speed, 0.0, stats.friction * delta)
	velocity = transform.x * speed
	move_and_slide()
	if spinout_timer <= 0.0:
		is_spinning_out = false

func take_damage(amount: int = 1) -> void:
	hp -= amount
	if hp <= 0:
		_on_wrecked()

func _on_wrecked() -> void:
	visible = false
	set_physics_process(false)
	# Respawn handled by game manager after 3 seconds

func respawn(spawn_position: Vector2, spawn_rotation: float) -> void:
	global_position = spawn_position
	rotation = spawn_rotation
	speed = 0.0
	hp = 3
	is_spinning_out = false
	visible = true
	set_physics_process(true)
```

- [ ] **Step 4: Create car.tscn scene**

```
[gd_scene load_steps=3 format=3 uid="uid://car_scene"]

[ext_resource type="Script" path="res://scripts/car/car_body.gd" id="1"]
[ext_resource type="Script" path="res://scripts/car/car_visual.gd" id="2"]

[node name="Car" type="CharacterBody2D"]
collision_layer = 1
collision_mask = 3
script = ExtResource("1")

[node name="Sprite" type="Sprite2D" parent="."]
script = ExtResource("2")

[node name="Collision" type="CollisionShape2D" parent="."]
shape = SubResource("RectangleShape2D_car")

[sub_resource type="RectangleShape2D" id="RectangleShape2D_car"]
size = Vector2(40, 20)
```

Note: The actual sprite texture will be assigned later when art assets are created. For now, a white rectangle works as a placeholder.

- [ ] **Step 5: Test — open scene, drive car around**

1. Open `scenes/car/car.tscn` in Godot editor
2. Add a temporary Camera2D as child
3. Set `is_local_player = true` in the inspector or via a test script
4. Press F5 to run the scene
5. Use WASD to drive — car should accelerate, steer, brake, and decelerate with friction

Expected: Car moves smoothly, steers at speed, stops when no input.

- [ ] **Step 6: Commit**

```bash
git add scripts/car/ scenes/car/
git commit -m "feat: add car physics with steering, acceleration, drift, and spinout"
```

---

### Task 4: Track System + Oval Track

**Files:**
- Create: `scripts/track/track_data.gd`
- Create: `scripts/track/checkpoint.gd`
- Create: `scenes/track/checkpoint.tscn`
- Create: `scenes/track/track_oval.tscn`

- [ ] **Step 1: Create track_data.gd resource**

```gdscript
# scripts/track/track_data.gd
extends Resource
class_name TrackData

@export var track_name := "Unnamed Track"
@export var spawn_points: Array[Vector2] = []
@export var spawn_rotations: Array[float] = []
@export var checkpoint_positions: Array[Vector2] = []
@export var checkpoint_rotations: Array[float] = []
@export var checkpoint_sizes: Array[Vector2] = []
@export var weapon_spawn_positions: Array[Vector2] = []
@export var ramp_positions: Array[Vector2] = []
@export var ramp_rotations: Array[float] = []
@export var total_checkpoints := 0
```

- [ ] **Step 2: Create checkpoint.gd**

```gdscript
# scripts/track/checkpoint.gd
extends Area2D
class_name Checkpoint

@export var checkpoint_index := 0
var is_finish_line := false

signal car_crossed(peer_id: int, checkpoint_index: int)

func _ready() -> void:
	body_entered.connect(_on_body_entered)
	is_finish_line = (checkpoint_index == 0)

func _on_body_entered(body: Node2D) -> void:
	if body is CarBody:
		car_crossed.emit(body.peer_id, checkpoint_index)
```

- [ ] **Step 3: Create checkpoint.tscn**

```
[gd_scene load_steps=2 format=3 uid="uid://checkpoint_scene"]

[ext_resource type="Script" path="res://scripts/track/checkpoint.gd" id="1"]

[node name="Checkpoint" type="Area2D"]
collision_layer = 0
collision_mask = 1
script = ExtResource("1")

[node name="CollisionShape" type="CollisionShape2D" parent="."]
shape = SubResource("RectangleShape2D_cp")

[sub_resource type="RectangleShape2D" id="RectangleShape2D_cp"]
size = Vector2(20, 200)
```

- [ ] **Step 4: Build oval track scene**

Create `scenes/track/track_oval.tscn` in the Godot editor:

1. Root node: `Node2D` named "TrackOval"
2. Add `TileMapLayer` child for road tiles (draw a simple oval road using placeholder grey tiles)
3. Add `StaticBody2D` children for outer/inner barriers with `CollisionPolygon2D` matching the track boundaries
4. Add 6 `Checkpoint` instances (from checkpoint.tscn) placed around the oval:
   - Checkpoint 0 (finish line) at the start/finish straight
   - Checkpoints 1-5 spaced evenly around the track
5. Add 4 spawn points as `Marker2D` nodes at the start grid (2x2 grid formation)
6. Add 3 `Marker2D` nodes for weapon pickup spawn locations

Track dimensions: approximately 3000x2000 pixels oval.

For now, use simple colored rectangles (ColorRect or draw calls) for the road surface — art comes later.

- [ ] **Step 5: Create track_oval_data.tres**

After placing nodes in the editor, create `resources/tracks/track_oval_data.tres`:

```gdscript
# Create via editor or script:
var data = TrackData.new()
data.track_name = "Classic Oval"
data.spawn_points = [
	Vector2(400, 950), Vector2(400, 1050),
	Vector2(300, 950), Vector2(300, 1050),
	Vector2(200, 950), Vector2(200, 1050),
	Vector2(100, 950), Vector2(100, 1050),
	Vector2(0, 950), Vector2(0, 1050),
]
data.spawn_rotations = [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
data.total_checkpoints = 6
ResourceSaver.save(data, "res://resources/tracks/track_oval_data.tres")
```

- [ ] **Step 6: Test — car drives around oval**

1. Add car scene as child of TrackOval
2. Position car at first spawn point
3. Run scene — car should be confined by barriers, checkpoints should trigger on crossing

Expected: Car stays on track, cannot drive through barriers, checkpoint signals fire.

- [ ] **Step 7: Commit**

```bash
git add scripts/track/ scenes/track/ resources/tracks/
git commit -m "feat: add track system with checkpoints and oval track"
```

---

### Task 5: Lap Counting + Race Flow

**Files:**
- Create: `scripts/track/race_controller.gd`
- Modify: `scripts/autoload/game_manager.gd`

- [ ] **Step 1: Create race_controller.gd**

```gdscript
# scripts/track/race_controller.gd
extends Node
class_name RaceController

signal lap_completed(peer_id: int, lap_number: int)
signal race_finished(peer_id: int, finish_time: float)
signal countdown_tick(seconds_left: int)
signal race_started()

var checkpoints_per_player: Dictionary = {}  # peer_id -> last_checkpoint_index
var race_timer := 0.0
var race_active := false
var countdown_active := false
var countdown_remaining := 3.0
var total_checkpoints := 0
var total_laps := 3

func setup(track_data: TrackData, laps: int) -> void:
	total_checkpoints = track_data.total_checkpoints
	total_laps = laps
	checkpoints_per_player.clear()
	race_timer = 0.0
	race_active = false

func register_car(peer_id: int) -> void:
	checkpoints_per_player[peer_id] = 0

func start_countdown() -> void:
	countdown_active = true
	countdown_remaining = 3.0
	GameManager.change_state(GameManager.GameState.COUNTDOWN)

func _process(delta: float) -> void:
	if countdown_active:
		var prev_sec = ceili(countdown_remaining)
		countdown_remaining -= delta
		var curr_sec = ceili(countdown_remaining)
		if curr_sec != prev_sec and curr_sec > 0:
			countdown_tick.emit(curr_sec)
		if countdown_remaining <= 0.0:
			countdown_active = false
			race_active = true
			race_started.emit()
			GameManager.change_state(GameManager.GameState.RACING)

	if race_active:
		race_timer += delta

func on_checkpoint_crossed(peer_id: int, checkpoint_index: int) -> void:
	if not race_active:
		return
	if not checkpoints_per_player.has(peer_id):
		return

	var last_cp = checkpoints_per_player[peer_id]
	var expected_next = (last_cp + 1) % total_checkpoints

	if checkpoint_index != expected_next:
		return  # Wrong checkpoint — ignore (prevents shortcutting)

	checkpoints_per_player[peer_id] = checkpoint_index

	# Crossed finish line (checkpoint 0) after hitting all others
	if checkpoint_index == 0 and last_cp == total_checkpoints - 1:
		var player = GameManager.players.get(peer_id)
		if player == null:
			return
		player["lap"] += 1
		lap_completed.emit(peer_id, player["lap"])

		if player["lap"] >= total_laps:
			player["finished"] = true
			player["finish_time"] = race_timer
			race_finished.emit(peer_id, race_timer)
			_check_all_finished()

func _check_all_finished() -> void:
	for peer_id in GameManager.players:
		if not GameManager.players[peer_id]["finished"]:
			return
	# All players finished
	race_active = false
	GameManager.change_state(GameManager.GameState.RESULTS)
```

- [ ] **Step 2: Wire checkpoints to race controller**

Add this to the track scene's root script (or create a small `track_manager.gd`):

```gdscript
# Add to track scene root or create scripts/track/track_manager.gd
extends Node2D

@onready var race_controller: RaceController = $RaceController

func _ready() -> void:
	for checkpoint in get_tree().get_nodes_in_group("checkpoints"):
		checkpoint.car_crossed.connect(race_controller.on_checkpoint_crossed)
```

Make sure each Checkpoint node in the track scene is added to group "checkpoints" (set in editor or in `checkpoint.gd` `_ready()`):

Add to `checkpoint.gd` `_ready()`:
```gdscript
func _ready() -> void:
	add_to_group("checkpoints")
	body_entered.connect(_on_body_entered)
	is_finish_line = (checkpoint_index == 0)
```

- [ ] **Step 3: Test — drive 3 laps, race ends**

1. Place car at spawn, set up race controller with 3 laps
2. Drive around oval crossing all checkpoints in order
3. Verify `lap_completed` fires after each lap
4. After 3 laps, verify `race_finished` fires and game state changes to RESULTS

Expected: Lap counter increments correctly, skipping checkpoints doesn't count.

- [ ] **Step 4: Commit**

```bash
git add scripts/track/race_controller.gd scripts/track/checkpoint.gd
git commit -m "feat: add race controller with lap counting, countdown, and finish detection"
```

---

### Task 6: Weapon System

**Files:**
- Create: `scripts/weapons/weapon_base.gd`
- Create: `scripts/weapons/missile.gd`
- Create: `scripts/weapons/oil_slick.gd`
- Create: `scripts/weapons/shield.gd`
- Create: `scripts/track/weapon_pickup.gd`
- Create: `scenes/weapons/missile.tscn`
- Create: `scenes/weapons/oil_slick.tscn`
- Create: `scenes/weapons/shield.tscn`
- Create: `scenes/track/weapon_pickup.tscn`

- [ ] **Step 1: Create weapon_base.gd**

```gdscript
# scripts/weapons/weapon_base.gd
extends Area2D
class_name WeaponBase

@export var weapon_name := "base"
@export var lifetime := 5.0

var owner_peer_id := 0
var timer := 0.0

func _ready() -> void:
	body_entered.connect(_on_body_entered)

func _process(delta: float) -> void:
	timer += delta
	if timer >= lifetime:
		queue_free()

func _on_body_entered(body: Node2D) -> void:
	if body is CarBody and body.peer_id != owner_peer_id:
		apply_effect(body)
		queue_free()

func apply_effect(_target: CarBody) -> void:
	pass  # Override in subclasses
```

- [ ] **Step 2: Create missile.gd**

```gdscript
# scripts/weapons/missile.gd
extends WeaponBase

@export var speed := 800.0
@export var homing_strength := 1.5
@export var damage := 1

var direction := Vector2.RIGHT
var target: CarBody = null

func _ready() -> void:
	super._ready()
	weapon_name = "missile"
	lifetime = 4.0
	collision_layer = 16  # projectiles layer
	collision_mask = 1    # cars layer

func fire(from_position: Vector2, from_rotation: float, shooter_peer_id: int) -> void:
	global_position = from_position
	rotation = from_rotation
	direction = Vector2.RIGHT.rotated(from_rotation)
	owner_peer_id = shooter_peer_id
	_find_nearest_target()

func _process(delta: float) -> void:
	super._process(delta)
	if target and is_instance_valid(target) and target.peer_id != owner_peer_id:
		var to_target = (target.global_position - global_position).normalized()
		direction = direction.lerp(to_target, homing_strength * delta).normalized()
		rotation = direction.angle()
	global_position += direction * speed * delta

func _find_nearest_target() -> void:
	var nearest_dist := 99999.0
	for car in get_tree().get_nodes_in_group("cars"):
		if car is CarBody and car.peer_id != owner_peer_id:
			var dist = global_position.distance_to(car.global_position)
			# Only target cars ahead (within 90 degrees of firing direction)
			var to_car = (car.global_position - global_position).normalized()
			if direction.dot(to_car) > 0.0 and dist < nearest_dist:
				nearest_dist = dist
				target = car

func apply_effect(target_car: CarBody) -> void:
	target_car.take_damage(damage)
	target_car.apply_spinout()
```

- [ ] **Step 3: Create oil_slick.gd**

```gdscript
# scripts/weapons/oil_slick.gd
extends WeaponBase

func _ready() -> void:
	super._ready()
	weapon_name = "oil_slick"
	lifetime = 15.0
	collision_layer = 16
	collision_mask = 1

func drop(from_position: Vector2, from_rotation: float, dropper_peer_id: int) -> void:
	# Drop behind the car
	var behind = Vector2.LEFT.rotated(from_rotation) * 60.0
	global_position = from_position + behind
	owner_peer_id = dropper_peer_id

func apply_effect(target_car: CarBody) -> void:
	target_car.apply_spinout()
```

- [ ] **Step 4: Create shield.gd**

```gdscript
# scripts/weapons/shield.gd
extends Node2D
class_name ShieldEffect

var active := false
var owner_car: CarBody = null
var duration := 10.0
var timer := 0.0

func activate(car: CarBody) -> void:
	owner_car = car
	active = true
	timer = 0.0
	# Visual: add a circle around the car
	visible = true

func _process(delta: float) -> void:
	if not active:
		return
	timer += delta
	if timer >= duration:
		deactivate()
		return
	if owner_car and is_instance_valid(owner_car):
		global_position = owner_car.global_position

func block_hit() -> bool:
	if active:
		deactivate()
		return true
	return false

func deactivate() -> void:
	active = false
	visible = false
	queue_free()
```

Update `car_body.gd` `take_damage()` to check shield:

```gdscript
# Add to car_body.gd
var shield_node: ShieldEffect = null

func take_damage(amount: int = 1) -> void:
	if shield_node and shield_node.block_hit():
		shield_node = null
		return
	hp -= amount
	if hp <= 0:
		_on_wrecked()
```

- [ ] **Step 5: Create weapon_pickup.gd**

```gdscript
# scripts/track/weapon_pickup.gd
extends Area2D
class_name WeaponPickup

const WEAPONS := ["missile", "oil_slick", "shield"]
const RESPAWN_TIME := 10.0

var active := true
var respawn_timer := 0.0

@onready var sprite: Sprite2D = $Sprite
@onready var collision: CollisionShape2D = $CollisionShape

func _ready() -> void:
	add_to_group("weapon_pickups")
	body_entered.connect(_on_body_entered)

func _process(delta: float) -> void:
	if not active:
		respawn_timer -= delta
		if respawn_timer <= 0.0:
			_respawn()

func _on_body_entered(body: Node2D) -> void:
	if not active:
		return
	if body is CarBody and body.current_weapon == "":
		var weapon = WEAPONS.pick_random()
		body.current_weapon = weapon
		_deactivate()

func _deactivate() -> void:
	active = false
	visible = false
	collision.set_deferred("disabled", true)
	respawn_timer = RESPAWN_TIME

func _respawn() -> void:
	active = true
	visible = true
	collision.set_deferred("disabled", false)
```

- [ ] **Step 6: Add weapon firing to car_body.gd**

Add to `car_body.gd`:

```gdscript
# Add these constants at top of car_body.gd
const MissileScene = preload("res://scenes/weapons/missile.tscn")
const OilSlickScene = preload("res://scenes/weapons/oil_slick.tscn")
const ShieldScene = preload("res://scenes/weapons/shield.tscn")

# Add to _physics_process before move_and_slide:
func _physics_process(delta: float) -> void:
	if is_spinning_out:
		_process_spinout(delta)
		return

	if is_local_player:
		_read_input()
		if Input.is_action_just_pressed("fire_weapon"):
			fire_weapon()

	_apply_steering(delta)
	_apply_acceleration(delta)
	_apply_friction(delta)

	velocity = transform.x * speed
	move_and_slide()

func fire_weapon() -> void:
	if current_weapon == "":
		return

	match current_weapon:
		"missile":
			var missile = MissileScene.instantiate()
			get_parent().add_child(missile)
			missile.fire(global_position + transform.x * 30, rotation, peer_id)
		"oil_slick":
			var slick = OilSlickScene.instantiate()
			get_parent().add_child(slick)
			slick.drop(global_position, rotation, peer_id)
		"shield":
			var shield = ShieldScene.instantiate()
			add_child(shield)
			shield.activate(self)
			shield_node = shield

	current_weapon = ""
```

- [ ] **Step 7: Create weapon scenes (missile.tscn, oil_slick.tscn, shield.tscn)**

`scenes/weapons/missile.tscn`:
```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/weapons/missile.gd" id="1"]

[node name="Missile" type="Area2D"]
collision_layer = 16
collision_mask = 1
script = ExtResource("1")

[node name="Sprite" type="Sprite2D" parent="."]

[node name="CollisionShape" type="CollisionShape2D" parent="."]
shape = SubResource("CircleShape2D_missile")

[sub_resource type="CircleShape2D" id="CircleShape2D_missile"]
radius = 8.0
```

`scenes/weapons/oil_slick.tscn`:
```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/weapons/oil_slick.gd" id="1"]

[node name="OilSlick" type="Area2D"]
collision_layer = 16
collision_mask = 1
script = ExtResource("1")

[node name="Sprite" type="Sprite2D" parent="."]

[node name="CollisionShape" type="CollisionShape2D" parent="."]
shape = SubResource("CircleShape2D_oil")

[sub_resource type="CircleShape2D" id="CircleShape2D_oil"]
radius = 20.0
```

`scenes/weapons/shield.tscn`:
```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/weapons/shield.gd" id="1"]

[node name="Shield" type="Node2D"]
script = ExtResource("1")

[node name="Sprite" type="Sprite2D" parent="."]
```

- [ ] **Step 8: Create weapon_pickup.tscn**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/track/weapon_pickup.gd" id="1"]

[node name="WeaponPickup" type="Area2D"]
collision_layer = 8
collision_mask = 1
script = ExtResource("1")

[node name="Sprite" type="Sprite2D" parent="."]

[node name="CollisionShape" type="CollisionShape2D" parent="."]
shape = SubResource("CircleShape2D_pickup")

[sub_resource type="CircleShape2D" id="CircleShape2D_pickup"]
radius = 25.0
```

- [ ] **Step 9: Test — pick up weapon, fire at wall**

1. Place car + weapon pickup on track
2. Drive over pickup — car should receive a random weapon
3. Press Space (fire_weapon) — missile should fly forward, oil slick should drop behind, shield should surround car
4. Fire missile at second test car — target should spin out

Expected: Weapons fire correctly, pickups disappear and respawn after 10s.

- [ ] **Step 10: Commit**

```bash
git add scripts/weapons/ scenes/weapons/ scripts/track/weapon_pickup.gd scenes/track/weapon_pickup.tscn
git commit -m "feat: add weapon system with missile, oil slick, shield, and pickups"
```

---

### Task 7: Stunt System + Boost Meter

**Files:**
- Create: `scripts/car/car_stunt.gd`
- Modify: `scripts/car/car_body.gd`

- [ ] **Step 1: Create car_stunt.gd**

```gdscript
# scripts/car/car_stunt.gd
extends Node
class_name CarStunt

signal boost_changed(value: float)
signal stunt_landed(stunt_name: String, multiplier: float)
signal combo_started()
signal combo_broken()

const MAX_BOOST := 100.0
const BOOST_SPEED_MULT := 1.6
const BOOST_DRAIN_RATE := 40.0  # per second
const DRIFT_FILL_RATE := 15.0   # per second while drifting
const BARREL_ROLL_FILL := 25.0
const COMBO_WINDOW := 2.0       # seconds to chain tricks

var boost_meter := 0.0
var is_boosting := false
var combo_count := 0
var combo_timer := 0.0
var is_in_air := false
var air_rotation := 0.0
var barrel_roll_threshold := PI * 1.8  # ~full rotation

var car: CarBody = null

func setup(car_body: CarBody) -> void:
	car = car_body

func _process(delta: float) -> void:
	if car == null:
		return

	_process_drift(delta)
	_process_air_stunt(delta)
	_process_boost(delta)
	_process_combo(delta)

func _process_drift(delta: float) -> void:
	if car.is_drifting and abs(car.speed) > 100.0:
		var fill = DRIFT_FILL_RATE * _get_combo_mult() * delta
		add_boost(fill)

func _process_air_stunt(_delta: float) -> void:
	if is_in_air:
		air_rotation += abs(car.steer_input) * 8.0 * _delta
		if air_rotation >= barrel_roll_threshold:
			_land_stunt("Barrel Roll", BARREL_ROLL_FILL)
			air_rotation = 0.0

func _process_boost(delta: float) -> void:
	if is_boosting and boost_meter > 0.0:
		boost_meter -= BOOST_DRAIN_RATE * delta
		if boost_meter <= 0.0:
			boost_meter = 0.0
			is_boosting = false
		boost_changed.emit(boost_meter)
	elif is_boosting:
		is_boosting = false

func _process_combo(delta: float) -> void:
	if combo_count > 0:
		combo_timer -= delta
		if combo_timer <= 0.0:
			combo_count = 0
			combo_broken.emit()

func activate_boost() -> void:
	if boost_meter >= 20.0:  # minimum threshold
		is_boosting = true

func deactivate_boost() -> void:
	is_boosting = false

func add_boost(amount: float) -> void:
	boost_meter = clampf(boost_meter + amount, 0.0, MAX_BOOST)
	boost_changed.emit(boost_meter)

func enter_air() -> void:
	is_in_air = true
	air_rotation = 0.0

func exit_air() -> void:
	if is_in_air and air_rotation > 0.0 and air_rotation < barrel_roll_threshold:
		# Failed stunt — crash penalty
		boost_meter = clampf(boost_meter - 10.0, 0.0, MAX_BOOST)
		combo_count = 0
		combo_broken.emit()
	is_in_air = false
	air_rotation = 0.0

func start_drift() -> void:
	car.is_drifting = true

func stop_drift() -> void:
	car.is_drifting = false

func _land_stunt(stunt_name: String, fill_amount: float) -> void:
	combo_count += 1
	combo_timer = COMBO_WINDOW
	if combo_count == 1:
		combo_started.emit()
	var mult = _get_combo_mult()
	add_boost(fill_amount * mult)
	stunt_landed.emit(stunt_name, mult)

func _get_combo_mult() -> float:
	if combo_count >= 3:
		return 2.0
	elif combo_count >= 2:
		return 1.5
	return 1.0

func get_speed_multiplier() -> float:
	if is_boosting:
		return BOOST_SPEED_MULT
	return 1.0
```

- [ ] **Step 2: Integrate stunt system into car_body.gd**

Add to `car_body.gd`:

```gdscript
# Add at top of car_body.gd
var stunt: CarStunt = null

# Add to _ready()
func _ready() -> void:
	if stats == null:
		stats = CarStats.new()
	add_to_group("cars")
	stunt = CarStunt.new()
	add_child(stunt)
	stunt.setup(self)

# Modify _read_input to handle drift and boost
func _read_input() -> void:
	is_accelerating = Input.is_action_pressed("accelerate")
	is_braking = Input.is_action_pressed("brake")
	steer_input = Input.get_axis("steer_left", "steer_right")

	# Drift: brake + steer while moving
	if is_braking and abs(steer_input) > 0.3 and abs(speed) > 100.0:
		if not is_drifting:
			stunt.start_drift()
	elif is_drifting:
		stunt.stop_drift()

	# Boost activation
	if Input.is_action_just_pressed("activate_boost"):
		stunt.activate_boost()
	if Input.is_action_just_released("activate_boost"):
		stunt.deactivate_boost()

# Modify _apply_acceleration to include boost
func _apply_acceleration(delta: float) -> void:
	var speed_mult = stunt.get_speed_multiplier() if stunt else 1.0
	var effective_max = stats.max_speed * speed_mult
	if is_accelerating:
		speed = move_toward(speed, effective_max, stats.acceleration * delta)
	elif is_braking:
		speed = move_toward(speed, -stats.max_speed * 0.3, stats.brake_force * delta)
```

- [ ] **Step 3: Test — drift to fill boost, activate boost**

1. Drive car at speed, hold brake + steer to drift
2. Boost meter should fill while drifting
3. Press Shift to activate boost — car should accelerate beyond normal max speed
4. Release Shift — boost drains, car returns to normal speed

Expected: Drift fills meter at ~15/sec, boost gives 1.6x speed.

- [ ] **Step 4: Commit**

```bash
git add scripts/car/car_stunt.gd scripts/car/car_body.gd
git commit -m "feat: add stunt system with drift, barrel roll, combo multiplier, and boost"
```

---

### Task 8: Multiplayer Sync

**Files:**
- Create: `scripts/network/player_input.gd`
- Create: `scripts/network/sync_manager.gd`
- Create: `scripts/network/host_authority.gd`

- [ ] **Step 1: Create player_input.gd**

```gdscript
# scripts/network/player_input.gd
extends Node
class_name PlayerInput

var input_data := {
	"accelerate": false,
	"brake": false,
	"steer": 0.0,
	"fire_weapon": false,
	"activate_boost": false,
	"deactivate_boost": false,
	"tick": 0,
}

var tick_counter := 0

func capture() -> Dictionary:
	tick_counter += 1
	input_data["accelerate"] = Input.is_action_pressed("accelerate")
	input_data["brake"] = Input.is_action_pressed("brake")
	input_data["steer"] = Input.get_axis("steer_left", "steer_right")
	input_data["fire_weapon"] = Input.is_action_just_pressed("fire_weapon")
	input_data["activate_boost"] = Input.is_action_just_pressed("activate_boost")
	input_data["deactivate_boost"] = Input.is_action_just_released("activate_boost")
	input_data["tick"] = tick_counter
	return input_data.duplicate()
```

- [ ] **Step 2: Create sync_manager.gd**

```gdscript
# scripts/network/sync_manager.gd
extends Node
class_name SyncManager

const SYNC_RATE := 1.0 / 60.0  # 60 times per second
const INTERPOLATION_SPEED := 15.0

var sync_timer := 0.0
var remote_car_states: Dictionary = {}  # peer_id -> { position, rotation, speed, ... }

func _process(delta: float) -> void:
	sync_timer += delta
	if sync_timer >= SYNC_RATE:
		sync_timer -= SYNC_RATE
		if NetworkManager.is_host:
			_broadcast_game_state()
		else:
			_send_input_to_host()

	_interpolate_remote_cars(delta)

func _broadcast_game_state() -> void:
	var state := {}
	for peer_id in GameManager.players:
		var player = GameManager.players[peer_id]
		var car: CarBody = player.get("car_node")
		if car == null:
			continue
		state[peer_id] = {
			"pos": car.global_position,
			"rot": car.rotation,
			"spd": car.speed,
			"hp": car.hp,
			"wpn": car.current_weapon,
			"bst": car.stunt.boost_meter if car.stunt else 0.0,
			"spin": car.is_spinning_out,
			"drft": car.is_drifting,
		}
	rpc("receive_game_state", state)

@rpc("authority", "unreliable")
func receive_game_state(state: Dictionary) -> void:
	remote_car_states = state

func _send_input_to_host() -> void:
	var local_car = _get_local_car()
	if local_car == null:
		return
	var input_node = local_car.get_node_or_null("PlayerInput")
	if input_node == null:
		return
	var input_data = input_node.capture()
	rpc_id(1, "receive_player_input", input_data)

@rpc("any_peer", "unreliable")
func receive_player_input(input_data: Dictionary) -> void:
	var sender_id = multiplayer.get_remote_sender_id()
	var player = GameManager.players.get(sender_id)
	if player == null:
		return
	var car: CarBody = player.get("car_node")
	if car == null:
		return
	# Apply remote player's input on host
	car.is_accelerating = input_data.get("accelerate", false)
	car.is_braking = input_data.get("brake", false)
	car.steer_input = input_data.get("steer", 0.0)
	if input_data.get("fire_weapon", false):
		car.fire_weapon()
	if input_data.get("activate_boost", false):
		car.stunt.activate_boost()
	if input_data.get("deactivate_boost", false):
		car.stunt.deactivate_boost()

func _interpolate_remote_cars(delta: float) -> void:
	var my_id = multiplayer.get_unique_id()
	for peer_id in remote_car_states:
		if peer_id == my_id:
			continue  # Don't interpolate local car
		var state = remote_car_states[peer_id]
		var player = GameManager.players.get(peer_id)
		if player == null:
			continue
		var car: CarBody = player.get("car_node")
		if car == null:
			continue
		car.global_position = car.global_position.lerp(state["pos"], INTERPOLATION_SPEED * delta)
		car.rotation = lerp_angle(car.rotation, state["rot"], INTERPOLATION_SPEED * delta)
		car.speed = state["spd"]
		car.hp = state["hp"]
		car.current_weapon = state["wpn"]
		car.is_spinning_out = state["spin"]
		car.is_drifting = state["drft"]
		if car.stunt:
			car.stunt.boost_meter = state["bst"]

func _get_local_car() -> CarBody:
	var my_id = multiplayer.get_unique_id()
	var player = GameManager.players.get(my_id)
	if player:
		return player.get("car_node")
	return null

# Reliable RPCs for game events
@rpc("authority", "reliable")
func weapon_fired(peer_id: int, weapon_name: String, pos: Vector2, rot: float) -> void:
	# All clients spawn the weapon visual
	var player = GameManager.players.get(peer_id)
	if player == null:
		return
	var car: CarBody = player.get("car_node")
	if car == null:
		return
	# Weapon already created on host via input — clients just see it via state sync

@rpc("authority", "reliable")
func player_hit(target_peer_id: int, damage: int) -> void:
	var player = GameManager.players.get(target_peer_id)
	if player == null:
		return
	var car: CarBody = player.get("car_node")
	if car:
		car.take_damage(damage)
		car.apply_spinout()

@rpc("authority", "reliable")
func lap_completed_sync(peer_id: int, lap: int) -> void:
	var player = GameManager.players.get(peer_id)
	if player:
		player["lap"] = lap

@rpc("authority", "reliable")
func race_finished_sync(peer_id: int, finish_time: float) -> void:
	var player = GameManager.players.get(peer_id)
	if player:
		player["finished"] = true
		player["finish_time"] = finish_time
```

- [ ] **Step 3: Create host_authority.gd**

```gdscript
# scripts/network/host_authority.gd
extends Node
class_name HostAuthority

# Validates inputs and enforces game rules on the host side
# Prevents basic cheating: speed hacks, teleportation, invalid weapon use

const MAX_SPEED_TOLERANCE := 1.2  # Allow 20% over max speed (boost can exceed)

func validate_car_state(car: CarBody) -> void:
	if car == null:
		return
	var max_allowed = car.stats.max_speed * CarStunt.BOOST_SPEED_MULT * MAX_SPEED_TOLERANCE
	if abs(car.speed) > max_allowed:
		car.speed = sign(car.speed) * car.stats.max_speed

func validate_weapon_fire(car: CarBody) -> bool:
	if car.current_weapon == "":
		return false
	if car.is_spinning_out:
		return false
	return true
```

- [ ] **Step 4: Test — two instances over LAN**

1. Export the project or run two editor instances
2. Instance 1: Create LAN game
3. Instance 2: Join via 127.0.0.1
4. Both cars should appear, drive around, and see each other's movement smoothly
5. Fire weapon from one car — other car should see it and take damage

Expected: Smooth 60fps state sync, weapon effects replicated, position interpolated.

- [ ] **Step 5: Commit**

```bash
git add scripts/network/
git commit -m "feat: add multiplayer sync with host-authoritative input relay and state interpolation"
```

---

### Task 9: UI — Main Menu + Lobby

**Files:**
- Create: `scripts/ui/main_menu.gd`
- Create: `scripts/ui/lobby_screen.gd`
- Create: `scenes/ui/main_menu.tscn`
- Create: `scenes/ui/lobby_screen.tscn`
- Modify: `scenes/main.tscn`

- [ ] **Step 1: Create main_menu.gd**

```gdscript
# scripts/ui/main_menu.gd
extends Control

@onready var name_input: LineEdit = $VBoxContainer/NameInput
@onready var host_button: Button = $VBoxContainer/HostButton
@onready var join_container: HBoxContainer = $VBoxContainer/JoinContainer
@onready var ip_input: LineEdit = $VBoxContainer/JoinContainer/IPInput
@onready var join_button: Button = $VBoxContainer/JoinContainer/JoinButton
@onready var status_label: Label = $VBoxContainer/StatusLabel

func _ready() -> void:
	host_button.pressed.connect(_on_host_pressed)
	join_button.pressed.connect(_on_join_pressed)
	NetworkManager.connection_succeeded.connect(_on_connected)
	NetworkManager.connection_failed.connect(_on_connect_failed)
	NetworkManager.server_created.connect(_on_server_created)
	name_input.text = "Player " + str(randi() % 1000)

func _on_host_pressed() -> void:
	NetworkManager.local_player_name = name_input.text
	NetworkManager.local_color_index = randi() % 3
	var error = NetworkManager.create_server()
	if error != OK:
		status_label.text = "Failed to create server: " + str(error)

func _on_join_pressed() -> void:
	var ip = ip_input.text.strip_edges()
	if ip == "":
		ip = "127.0.0.1"
	NetworkManager.local_player_name = name_input.text
	NetworkManager.local_color_index = randi() % 3
	status_label.text = "Connecting to " + ip + "..."
	var error = NetworkManager.join_server(ip)
	if error != OK:
		status_label.text = "Failed to connect: " + str(error)

func _on_server_created() -> void:
	_go_to_lobby()

func _on_connected() -> void:
	_go_to_lobby()

func _on_connect_failed() -> void:
	status_label.text = "Connection failed. Check IP and try again."

func _go_to_lobby() -> void:
	GameManager.change_state(GameManager.GameState.LOBBY)
	get_tree().change_scene_to_file("res://scenes/ui/lobby_screen.tscn")
```

- [ ] **Step 2: Create lobby_screen.gd**

```gdscript
# scripts/ui/lobby_screen.gd
extends Control

@onready var player_list: VBoxContainer = $VBoxContainer/PlayerList
@onready var start_button: Button = $VBoxContainer/StartButton
@onready var back_button: Button = $VBoxContainer/BackButton
@onready var status_label: Label = $VBoxContainer/StatusLabel
@onready var track_selector: OptionButton = $VBoxContainer/TrackSelector

func _ready() -> void:
	start_button.pressed.connect(_on_start_pressed)
	back_button.pressed.connect(_on_back_pressed)
	NetworkManager.player_connected.connect(_on_player_changed)
	NetworkManager.player_disconnected.connect(_on_player_changed)
	start_button.visible = NetworkManager.is_host
	track_selector.visible = NetworkManager.is_host
	track_selector.add_item("Classic Oval", 0)
	track_selector.add_item("Ramps & Obstacles", 1)
	_refresh_player_list()

func _on_player_changed(_peer_id: int) -> void:
	_refresh_player_list()

func _refresh_player_list() -> void:
	for child in player_list.get_children():
		child.queue_free()
	for peer_id in GameManager.players:
		var p = GameManager.players[peer_id]
		var label = Label.new()
		var host_tag = " (Host)" if peer_id == 1 else ""
		label.text = p["name"] + host_tag
		label.add_theme_color_override("font_color", CarVisual.COLORS[p["color_index"]])
		player_list.add_child(label)
	status_label.text = str(GameManager.players.size()) + "/" + str(NetworkManager.MAX_PLAYERS) + " players"

func _on_start_pressed() -> void:
	if GameManager.players.size() < 1:
		return
	var track_idx = track_selector.selected
	var track_name = "track_oval" if track_idx == 0 else "track_ramps"
	GameManager.race_config["track"] = track_name
	rpc("load_race", track_name)
	load_race(track_name)

@rpc("authority", "reliable")
func load_race(track_name: String) -> void:
	GameManager.race_config["track"] = track_name
	GameManager.reset_race()
	get_tree().change_scene_to_file("res://scenes/track/" + track_name + ".tscn")

func _on_back_pressed() -> void:
	NetworkManager.disconnect_from_game()
	GameManager.change_state(GameManager.GameState.MENU)
	get_tree().change_scene_to_file("res://scenes/ui/main_menu.tscn")
```

- [ ] **Step 3: Create main_menu.tscn**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/main_menu.gd" id="1"]

[node name="MainMenu" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
script = ExtResource("1")

[node name="VBoxContainer" type="VBoxContainer" parent="."]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -150.0
offset_top = -120.0
offset_right = 150.0
offset_bottom = 120.0

[node name="TitleLabel" type="Label" parent="VBoxContainer"]
layout_mode = 2
text = "TURBO CLASH"
horizontal_alignment = 1

[node name="NameInput" type="LineEdit" parent="VBoxContainer"]
layout_mode = 2
placeholder_text = "Your name"

[node name="HostButton" type="Button" parent="VBoxContainer"]
layout_mode = 2
text = "Create LAN Game"

[node name="JoinContainer" type="HBoxContainer" parent="VBoxContainer"]
layout_mode = 2

[node name="IPInput" type="LineEdit" parent="VBoxContainer/JoinContainer"]
layout_mode = 2
size_flags_horizontal = 3
placeholder_text = "Host IP (e.g. 192.168.1.5)"

[node name="JoinButton" type="Button" parent="VBoxContainer/JoinContainer"]
layout_mode = 2
text = "Join"

[node name="StatusLabel" type="Label" parent="VBoxContainer"]
layout_mode = 2
horizontal_alignment = 1
```

- [ ] **Step 4: Create lobby_screen.tscn**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/lobby_screen.gd" id="1"]

[node name="LobbyScreen" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
script = ExtResource("1")

[node name="VBoxContainer" type="VBoxContainer" parent="."]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -150.0
offset_top = -150.0
offset_right = 150.0
offset_bottom = 150.0

[node name="LobbyTitle" type="Label" parent="VBoxContainer"]
layout_mode = 2
text = "LOBBY"
horizontal_alignment = 1

[node name="TrackSelector" type="OptionButton" parent="VBoxContainer"]
layout_mode = 2

[node name="PlayerList" type="VBoxContainer" parent="VBoxContainer"]
layout_mode = 2

[node name="StatusLabel" type="Label" parent="VBoxContainer"]
layout_mode = 2
horizontal_alignment = 1

[node name="StartButton" type="Button" parent="VBoxContainer"]
layout_mode = 2
text = "Start Race"

[node name="BackButton" type="Button" parent="VBoxContainer"]
layout_mode = 2
text = "Back"
```

- [ ] **Step 5: Update main.tscn to load main menu**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="PackedScene" path="res://scenes/ui/main_menu.tscn" id="1"]

[node name="Main" type="Node2D"]

[node name="MainMenu" parent="." instance=ExtResource("1")]
```

- [ ] **Step 6: Test — host, join, see lobby, start race**

1. Run project — main menu appears
2. Enter name, click "Create LAN Game" — goes to lobby
3. Second instance joins via IP — both see player list update
4. Host selects track, clicks Start — both load the track scene

Expected: Full flow from menu → lobby → track loads.

- [ ] **Step 7: Commit**

```bash
git add scripts/ui/ scenes/ui/ scenes/main.tscn
git commit -m "feat: add main menu and lobby UI with LAN host/join flow"
```

---

### Task 10: HUD + Results Screen

**Files:**
- Create: `scripts/ui/hud.gd`
- Create: `scripts/ui/results_screen.gd`
- Create: `scenes/ui/hud.tscn`
- Create: `scenes/ui/results_screen.tscn`

- [ ] **Step 1: Create hud.gd**

```gdscript
# scripts/ui/hud.gd
extends CanvasLayer

@onready var position_label: Label = $TopBar/PositionLabel
@onready var lap_label: Label = $TopBar/LapLabel
@onready var speed_label: Label = $TopBar/SpeedLabel
@onready var hp_container: HBoxContainer = $BottomBar/HPContainer
@onready var boost_bar: ProgressBar = $BottomBar/BoostBar
@onready var weapon_label: Label = $BottomBar/WeaponLabel
@onready var countdown_label: Label = $CenterContainer/CountdownLabel
@onready var combo_label: Label = $CenterContainer/ComboLabel

var local_car: CarBody = null
var combo_display_timer := 0.0

func setup(car: CarBody, race_controller: RaceController) -> void:
	local_car = car
	race_controller.countdown_tick.connect(_on_countdown_tick)
	race_controller.race_started.connect(_on_race_started)
	race_controller.lap_completed.connect(_on_lap_completed)
	if car.stunt:
		car.stunt.boost_changed.connect(_on_boost_changed)
		car.stunt.stunt_landed.connect(_on_stunt_landed)
		car.stunt.combo_broken.connect(_on_combo_broken)
	countdown_label.visible = false
	combo_label.visible = false
	boost_bar.max_value = CarStunt.MAX_BOOST

func _process(_delta: float) -> void:
	if local_car == null:
		return

	speed_label.text = str(int(abs(local_car.speed))) + " px/s"
	weapon_label.text = local_car.current_weapon.to_upper() if local_car.current_weapon != "" else "NO WEAPON"

	var my_id = multiplayer.get_unique_id()
	var player = GameManager.players.get(my_id)
	if player:
		lap_label.text = "Lap " + str(player["lap"] + 1) + "/" + str(GameManager.race_config["laps"])
		_update_hp(player["hp"])

	# Position ranking
	var rankings = GameManager.get_rankings()
	for i in range(rankings.size()):
		if rankings[i].get("name") == NetworkManager.local_player_name:
			position_label.text = _ordinal(i + 1)
			break

	if combo_display_timer > 0.0:
		combo_display_timer -= _delta
		if combo_display_timer <= 0.0:
			combo_label.visible = false

func _update_hp(hp: int) -> void:
	for i in range(hp_container.get_child_count()):
		hp_container.get_child(i).visible = i < hp

func _on_boost_changed(value: float) -> void:
	boost_bar.value = value

func _on_countdown_tick(seconds_left: int) -> void:
	countdown_label.visible = true
	countdown_label.text = str(seconds_left)

func _on_race_started() -> void:
	countdown_label.text = "GO!"
	await get_tree().create_timer(1.0).timeout
	countdown_label.visible = false

func _on_lap_completed(peer_id: int, lap: int) -> void:
	if peer_id == multiplayer.get_unique_id():
		countdown_label.visible = true
		countdown_label.text = "Lap " + str(lap)
		await get_tree().create_timer(1.5).timeout
		countdown_label.visible = false

func _on_stunt_landed(stunt_name: String, multiplier: float) -> void:
	combo_label.visible = true
	combo_label.text = stunt_name + " x" + str(multiplier)
	combo_display_timer = 2.0

func _on_combo_broken() -> void:
	combo_label.visible = false

func _ordinal(n: int) -> String:
	var suffix = "th"
	if n % 100 < 11 or n % 100 > 13:
		match n % 10:
			1: suffix = "st"
			2: suffix = "nd"
			3: suffix = "rd"
	return str(n) + suffix
```

- [ ] **Step 2: Create results_screen.gd**

```gdscript
# scripts/ui/results_screen.gd
extends Control

@onready var results_list: VBoxContainer = $VBoxContainer/ResultsList
@onready var continue_button: Button = $VBoxContainer/ContinueButton

func _ready() -> void:
	continue_button.pressed.connect(_on_continue)
	_populate_results()

func _populate_results() -> void:
	var rankings = GameManager.get_rankings()
	for i in range(rankings.size()):
		var p = rankings[i]
		var label = Label.new()
		var time_str = ""
		if p["finished"]:
			time_str = " — " + _format_time(p["finish_time"])
		label.text = str(i + 1) + ". " + p["name"] + time_str
		label.add_theme_color_override("font_color", CarVisual.COLORS[p["color_index"]])
		results_list.add_child(label)

func _format_time(seconds: float) -> String:
	var mins = int(seconds) / 60
	var secs = int(seconds) % 60
	var ms = int((seconds - int(seconds)) * 100)
	return "%d:%02d.%02d" % [mins, secs, ms]

func _on_continue() -> void:
	GameManager.change_state(GameManager.GameState.LOBBY)
	get_tree().change_scene_to_file("res://scenes/ui/lobby_screen.tscn")
```

- [ ] **Step 3: Create hud.tscn**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/hud.gd" id="1"]

[node name="HUD" type="CanvasLayer"]
script = ExtResource("1")

[node name="TopBar" type="HBoxContainer" parent="."]
anchors_preset = 10
anchor_right = 1.0
offset_bottom = 40.0

[node name="PositionLabel" type="Label" parent="TopBar"]
layout_mode = 2
text = "1st"

[node name="LapLabel" type="Label" parent="TopBar"]
layout_mode = 2
text = "Lap 1/3"

[node name="SpeedLabel" type="Label" parent="TopBar"]
layout_mode = 2
text = "0 px/s"

[node name="BottomBar" type="HBoxContainer" parent="."]
anchors_preset = 12
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_top = -50.0

[node name="HPContainer" type="HBoxContainer" parent="BottomBar"]
layout_mode = 2

[node name="Heart1" type="TextureRect" parent="BottomBar/HPContainer"]
layout_mode = 2

[node name="Heart2" type="TextureRect" parent="BottomBar/HPContainer"]
layout_mode = 2

[node name="Heart3" type="TextureRect" parent="BottomBar/HPContainer"]
layout_mode = 2

[node name="BoostBar" type="ProgressBar" parent="BottomBar"]
layout_mode = 2
custom_minimum_size = Vector2(200, 0)
max_value = 100.0
value = 0.0

[node name="WeaponLabel" type="Label" parent="BottomBar"]
layout_mode = 2
text = "NO WEAPON"

[node name="CenterContainer" type="CenterContainer" parent="."]
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5

[node name="CountdownLabel" type="Label" parent="CenterContainer"]
layout_mode = 2
text = "3"

[node name="ComboLabel" type="Label" parent="CenterContainer"]
layout_mode = 2
text = ""
```

- [ ] **Step 4: Create results_screen.tscn**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/results_screen.gd" id="1"]

[node name="ResultsScreen" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
script = ExtResource("1")

[node name="VBoxContainer" type="VBoxContainer" parent="."]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -150.0
offset_top = -150.0
offset_right = 150.0
offset_bottom = 150.0

[node name="Title" type="Label" parent="VBoxContainer"]
layout_mode = 2
text = "RACE RESULTS"
horizontal_alignment = 1

[node name="ResultsList" type="VBoxContainer" parent="VBoxContainer"]
layout_mode = 2

[node name="ContinueButton" type="Button" parent="VBoxContainer"]
layout_mode = 2
text = "Continue"
```

- [ ] **Step 5: Test — full race flow**

1. Start game → menu → host → lobby → start race
2. Countdown appears (3, 2, 1, GO!)
3. HUD shows position, lap, HP, boost, weapon
4. Complete 3 laps — results screen appears with rankings
5. Click Continue — returns to lobby

Expected: Complete game loop from lobby through race to results.

- [ ] **Step 6: Commit**

```bash
git add scripts/ui/hud.gd scripts/ui/results_screen.gd scenes/ui/hud.tscn scenes/ui/results_screen.tscn
git commit -m "feat: add in-race HUD and results screen for complete game loop"
```

---

### Task 11: Ramp + Second Track

**Files:**
- Create: `scripts/track/ramp.gd`
- Create: `scenes/track/ramp.tscn`
- Create: `scenes/track/track_ramps.tscn`

- [ ] **Step 1: Create ramp.gd**

```gdscript
# scripts/track/ramp.gd
extends Area2D
class_name Ramp

@export var launch_speed_mult := 1.3
@export var air_time := 1.0  # seconds of "air" (visual effect in top-down)

func _ready() -> void:
	body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node2D) -> void:
	if body is CarBody:
		_launch_car(body)

func _launch_car(car: CarBody) -> void:
	car.speed *= launch_speed_mult
	if car.stunt:
		car.stunt.enter_air()
	# Visual: scale car up slightly to simulate height
	var tween = create_tween()
	tween.tween_property(car, "scale", Vector2(1.2, 1.2), air_time * 0.3)
	tween.tween_property(car, "scale", Vector2(1.2, 1.2), air_time * 0.4)
	tween.tween_property(car, "scale", Vector2(1.0, 1.0), air_time * 0.3)
	tween.tween_callback(func(): 
		if car.stunt:
			car.stunt.exit_air()
	)
```

- [ ] **Step 2: Create ramp.tscn**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/track/ramp.gd" id="1"]

[node name="Ramp" type="Area2D"]
collision_layer = 32
collision_mask = 1
script = ExtResource("1")

[node name="Sprite" type="Sprite2D" parent="."]

[node name="CollisionShape" type="CollisionShape2D" parent="."]
shape = SubResource("RectangleShape2D_ramp")

[sub_resource type="RectangleShape2D" id="RectangleShape2D_ramp"]
size = Vector2(60, 80)
```

- [ ] **Step 3: Build track_ramps scene in editor**

Create `scenes/track/track_ramps.tscn`:

1. Root: `Node2D` named "TrackRamps"
2. Road layout: a winding track with two hairpin turns and a long straight
3. Place 4 ramps at key locations (before turns, on the straight)
4. Place 6 checkpoints around the track
5. Place 5 weapon pickup locations
6. Add barriers (StaticBody2D) along track edges
7. Place obstacles: 3 small barriers in the middle of the road as chicanes
8. Spawn points: 2x5 grid at start line

Track dimensions: approximately 4000x3000 pixels.

- [ ] **Step 4: Test — drive ramps track, hit ramps, do barrel rolls**

1. Load ramps track, drive into a ramp
2. Car should scale up (air visual), speed boost on launch
3. Steer during air to accumulate barrel roll rotation
4. Land successfully — boost meter fills
5. Fail to complete rotation — boost meter loses 10 points

Expected: Ramps launch cars, barrel rolls work, combo system chains with drift.

- [ ] **Step 5: Commit**

```bash
git add scripts/track/ramp.gd scenes/track/ramp.tscn scenes/track/track_ramps.tscn
git commit -m "feat: add ramps and second track with obstacles"
```

---

### Task 12: Placeholder Art + Audio

**Files:**
- Create: All files in `assets/sprites/`, `assets/ui/`, `assets/audio/`

- [ ] **Step 1: Create placeholder sprites programmatically**

Create a utility script `scripts/util/generate_placeholders.gd` to run once in the editor:

```gdscript
# Run this in the editor console or as a @tool script
# Creates minimal placeholder art so the game is visually playable

@tool
extends EditorScript

func _run() -> void:
	_create_rect_png("res://assets/sprites/car_body.png", Vector2(40, 20), Color.WHITE)
	_create_rect_png("res://assets/sprites/missile.png", Vector2(12, 6), Color.RED)
	_create_rect_png("res://assets/sprites/oil_slick.png", Vector2(30, 30), Color(0.2, 0.2, 0.2))
	_create_rect_png("res://assets/sprites/shield_bubble.png", Vector2(50, 50), Color(0.3, 0.7, 1.0, 0.4))
	_create_rect_png("res://assets/sprites/weapon_pickup_box.png", Vector2(30, 30), Color.YELLOW)
	_create_rect_png("res://assets/sprites/ramp.png", Vector2(60, 80), Color(0.6, 0.4, 0.2))
	_create_rect_png("res://assets/sprites/track_tile_road.png", Vector2(64, 64), Color(0.3, 0.3, 0.3))
	_create_rect_png("res://assets/sprites/track_tile_grass.png", Vector2(64, 64), Color(0.2, 0.6, 0.2))
	_create_rect_png("res://assets/sprites/track_tile_barrier.png", Vector2(64, 64), Color(0.8, 0.1, 0.1))
	_create_rect_png("res://assets/sprites/boost_particle.png", Vector2(8, 8), Color(1.0, 0.6, 0.0))
	_create_rect_png("res://assets/ui/hp_heart.png", Vector2(20, 20), Color.RED)
	_create_rect_png("res://assets/ui/boost_bar_fill.png", Vector2(200, 20), Color(0.0, 0.8, 1.0))
	_create_rect_png("res://assets/ui/weapon_slot_bg.png", Vector2(50, 50), Color(0.2, 0.2, 0.2, 0.7))
	_create_rect_png("res://assets/ui/position_badge.png", Vector2(40, 40), Color(1.0, 0.8, 0.0))
	print("Placeholders generated!")

func _create_rect_png(path: String, size: Vector2, color: Color) -> void:
	var img = Image.create(int(size.x), int(size.y), false, Image.FORMAT_RGBA8)
	img.fill(color)
	img.save_png(path)
```

- [ ] **Step 2: Assign sprites to scenes**

Open each `.tscn` in the editor and assign the placeholder textures to the Sprite2D nodes:
- `car.tscn` → `car_body.png`
- `missile.tscn` → `missile.png`
- `oil_slick.tscn` → `oil_slick.png`
- `shield.tscn` → `shield_bubble.png`
- `weapon_pickup.tscn` → `weapon_pickup_box.png`
- `ramp.tscn` → `ramp.png`
- HUD hearts → `hp_heart.png`

- [ ] **Step 3: Create placeholder audio**

For MVP, generate silent/minimal WAV files. These will be replaced with real audio later:

```bash
# Generate 0.5s silent WAV files as placeholders
for name in engine_loop missile_fire oil_splat shield_activate hit_impact drift_screech boost_whoosh countdown_beep race_finish; do
    sox -n -r 44100 -c 1 "assets/audio/${name}.wav" trim 0.0 0.5 2>/dev/null || \
    python3 -c "
import wave, struct
f = wave.open('assets/audio/${name}.wav', 'w')
f.setnchannels(1)
f.setsampwidth(2)
f.setframerate(44100)
f.writeframes(struct.pack('<' + 'h' * 22050, *([0] * 22050)))
f.close()
"
done
```

- [ ] **Step 4: Commit**

```bash
git add assets/ scripts/util/
git commit -m "feat: add placeholder sprites and audio for playable MVP"
```

---

### Task 13: Track Scene Wiring (Car Spawning + Full Game Loop)

**Files:**
- Create: `scripts/track/track_manager.gd`

- [ ] **Step 1: Create track_manager.gd — the glue script**

```gdscript
# scripts/track/track_manager.gd
extends Node2D

const CarScene = preload("res://scenes/car/car.tscn")
const HUDScene = preload("res://scenes/ui/hud.tscn")
const ResultsScene = preload("res://scenes/ui/results_screen.tscn")

@export var track_data: TrackData

@onready var race_controller: RaceController = $RaceController
@onready var sync_manager: SyncManager = $SyncManager

var cars: Dictionary = {}  # peer_id -> CarBody
var hud: Node = null

func _ready() -> void:
	_spawn_all_cars()
	_setup_checkpoints()
	_setup_hud()

	race_controller.setup(track_data, GameManager.race_config.get("laps", 3))
	race_controller.race_finished.connect(_on_player_finished)

	GameManager.state_changed.connect(_on_state_changed)

	# Host starts countdown after short delay
	if NetworkManager.is_host:
		await get_tree().create_timer(1.0).timeout
		race_controller.start_countdown()

func _spawn_all_cars() -> void:
	var spawn_index := 0
	for peer_id in GameManager.players:
		var car: CarBody = CarScene.instantiate()
		car.peer_id = peer_id
		car.is_local_player = (peer_id == multiplayer.get_unique_id())
		car.hp = 3
		car.current_weapon = ""

		if spawn_index < track_data.spawn_points.size():
			car.global_position = track_data.spawn_points[spawn_index]
		if spawn_index < track_data.spawn_rotations.size():
			car.rotation = track_data.spawn_rotations[spawn_index]

		var visual = car.get_node("Sprite") as CarVisual
		if visual:
			visual.set_car_color(GameManager.players[peer_id]["color_index"])

		add_child(car)
		cars[peer_id] = car
		GameManager.players[peer_id]["car_node"] = car
		race_controller.register_car(peer_id)
		spawn_index += 1

	# Camera follows local car
	var my_id = multiplayer.get_unique_id()
	if cars.has(my_id):
		var camera = Camera2D.new()
		camera.zoom = Vector2(0.8, 0.8)
		camera.position_smoothing_enabled = true
		camera.position_smoothing_speed = 8.0
		cars[my_id].add_child(camera)
		camera.make_current()

func _setup_checkpoints() -> void:
	for checkpoint in get_tree().get_nodes_in_group("checkpoints"):
		checkpoint.car_crossed.connect(race_controller.on_checkpoint_crossed)

func _setup_hud() -> void:
	var my_id = multiplayer.get_unique_id()
	if cars.has(my_id):
		hud = HUDScene.instantiate()
		add_child(hud)
		hud.setup(cars[my_id], race_controller)

func _on_player_finished(peer_id: int, finish_time: float) -> void:
	if NetworkManager.is_host:
		sync_manager.rpc("race_finished_sync", peer_id, finish_time)

func _on_state_changed(_old: GameManager.GameState, new: GameManager.GameState) -> void:
	if new == GameManager.GameState.RESULTS:
		if hud:
			hud.queue_free()
		# Show results after short delay
		await get_tree().create_timer(2.0).timeout
		var results = ResultsScene.instantiate()
		add_child(results)
```

- [ ] **Step 2: Add RaceController and SyncManager as child nodes to both track scenes**

In both `track_oval.tscn` and `track_ramps.tscn`, add:
1. Attach `track_manager.gd` to the root node
2. Add child node `RaceController` (type: Node, script: `race_controller.gd`)
3. Add child node `SyncManager` (type: Node, script: `sync_manager.gd`)
4. Set the `track_data` export on the root to the corresponding `.tres` resource

- [ ] **Step 3: Test — full multiplayer game**

1. Run two instances
2. Instance 1: Host game
3. Instance 2: Join
4. Host starts race → countdown → race → drive 3 laps → results
5. Both players see each other, weapons work across network, laps sync

Expected: Complete multiplayer race from lobby to results with all systems working.

- [ ] **Step 4: Commit**

```bash
git add scripts/track/track_manager.gd scenes/track/
git commit -m "feat: wire full game loop — car spawning, camera, HUD, and race flow"
```

---

### Task 14: Disconnect Handling + Host Migration

**Files:**
- Modify: `scripts/autoload/network_manager.gd`

- [ ] **Step 1: Add disconnect handling and host migration**

Add to `network_manager.gd`:

```gdscript
# Add to network_manager.gd

var disconnect_timers: Dictionary = {}  # peer_id -> Timer
const DISCONNECT_TIMEOUT := 15.0

func _on_peer_disconnected(peer_id: int) -> void:
	if is_host:
		# Start timeout — car goes AI autopilot
		var player = GameManager.players.get(peer_id)
		if player and player.get("car_node"):
			player["car_node"].is_local_player = false
			# Simple AI: just drive forward
			player["car_node"].is_accelerating = true
			player["car_node"].steer_input = 0.0

		var timer = get_tree().create_timer(DISCONNECT_TIMEOUT)
		timer.timeout.connect(func(): _remove_disconnected_player(peer_id))
		disconnect_timers[peer_id] = timer

	player_disconnected.emit(peer_id)

func _remove_disconnected_player(peer_id: int) -> void:
	var player = GameManager.players.get(peer_id)
	if player and player.get("car_node"):
		player["car_node"].queue_free()
	GameManager.remove_player(peer_id)
	disconnect_timers.erase(peer_id)

# Host migration: if host disconnects, next peer becomes host
func _on_server_disconnected() -> void:
	# Server (host) died — attempt migration
	var my_id = multiplayer.get_unique_id()
	var peer_ids = GameManager.players.keys()
	peer_ids.sort()
	# Lowest remaining peer_id becomes new host
	if peer_ids.size() > 0 and peer_ids[0] == my_id:
		# We are the new host — create a new server
		_become_new_host()
	else:
		# Wait for new host to set up
		pass

func _become_new_host() -> void:
	peer.close()
	peer = ENetMultiplayerPeer.new()
	var error = peer.create_server(DEFAULT_PORT, MAX_PLAYERS)
	if error != OK:
		return
	multiplayer.multiplayer_peer = peer
	is_host = true
	# Remove old host from players
	GameManager.remove_player(1)
```

- [ ] **Step 2: Connect server_disconnected signal**

Add to `_connect_signals()`:

```gdscript
func _connect_signals() -> void:
	multiplayer.peer_connected.connect(_on_peer_connected)
	multiplayer.peer_disconnected.connect(_on_peer_disconnected)
	multiplayer.connected_to_server.connect(_on_connected_to_server)
	multiplayer.connection_failed.connect(_on_connection_failed)
	multiplayer.server_disconnected.connect(_on_server_disconnected)
```

- [ ] **Step 3: Test — disconnect during race**

1. Start 2-player race
2. Close player 2's window — player 2's car should continue driving forward (AI)
3. After 15 seconds, car disappears
4. Close host window — player 2 should attempt host migration

Expected: Graceful disconnect handling, no crashes.

- [ ] **Step 4: Commit**

```bash
git add scripts/autoload/network_manager.gd
git commit -m "feat: add disconnect timeout with AI autopilot and basic host migration"
```

---

### Task 15: Final Integration Test + Polish

**Files:**
- Modify: various scenes for visual polish

- [ ] **Step 1: Full integration test checklist**

Run through this complete test manually:

- [ ] Main menu loads on launch
- [ ] Can type player name
- [ ] Host creates LAN game → goes to lobby
- [ ] Second player joins via IP → both see updated player list
- [ ] Host selects track and starts race
- [ ] Countdown (3, 2, 1, GO!) appears on both screens
- [ ] Cars spawn at grid positions with correct colors
- [ ] WASD controls car (accelerate, brake, steer)
- [ ] Cars confined by track barriers
- [ ] Checkpoints trigger correctly — no skipping
- [ ] Lap counter increments after passing all checkpoints + finish line
- [ ] Weapon pickups: drive over → receive random weapon
- [ ] Missile: fires forward, slight homing, target spins out
- [ ] Oil slick: drops behind car, other cars spin out on contact
- [ ] Shield: blocks one hit, visual shows around car
- [ ] Drift: brake + steer fills boost meter
- [ ] Boost: Shift activates, car goes faster, meter drains
- [ ] Ramp: car scales up (air), barrel roll possible, boost fills on land
- [ ] HUD: position, lap, speed, HP, boost bar, weapon all update
- [ ] After 3 laps: race ends, results screen shows rankings with times
- [ ] Continue button returns to lobby
- [ ] Disconnect: car goes autopilot, removed after 15s
- [ ] Multiplayer: both players see each other's movement smoothly

- [ ] **Step 2: Fix any bugs found during integration test**

Address issues discovered in step 1. Common things to check:
- Collision layers/masks aligned correctly
- RPC functions called with correct authority
- Scene tree references valid after scene changes

- [ ] **Step 3: Final commit**

```bash
git add -A
git commit -m "feat: complete MVP — playable multiplayer top-down combat racing"
```

- [ ] **Step 4: Push to remote**

```bash
git push origin master
```

---

## Summary

| Task | Description | Key Files |
|------|-------------|-----------|
| 1 | Project scaffold | project.godot, game_manager.gd |
| 2 | Network manager (ENet LAN) | network_manager.gd |
| 3 | Car physics | car_body.gd, car_stats.gd, car.tscn |
| 4 | Track system + oval | track_data.gd, checkpoint.gd, track_oval.tscn |
| 5 | Lap counting + race flow | race_controller.gd |
| 6 | Weapon system | missile.gd, oil_slick.gd, shield.gd, weapon_pickup.gd |
| 7 | Stunt system + boost | car_stunt.gd |
| 8 | Multiplayer sync | sync_manager.gd, player_input.gd, host_authority.gd |
| 9 | Main menu + lobby UI | main_menu.gd, lobby_screen.gd |
| 10 | HUD + results screen | hud.gd, results_screen.gd |
| 11 | Ramps + second track | ramp.gd, track_ramps.tscn |
| 12 | Placeholder art + audio | assets/*  |
| 13 | Track scene wiring | track_manager.gd |
| 14 | Disconnect handling | network_manager.gd updates |
| 15 | Integration test + polish | Full test pass |
