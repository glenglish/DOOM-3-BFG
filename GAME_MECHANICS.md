# DOOM 3 BFG Edition — Game Mechanics for Java Developers

This document explains how DOOM 3 BFG Edition works internally, written for developers
familiar with Java. It maps C++ engine concepts to Java equivalents where possible.

---

## Table of Contents

1. [Repository Layout](#1-repository-layout)
2. [Architecture Overview](#2-architecture-overview)
3. [The Type / Reflection System](#3-the-type--reflection-system)
4. [The Entity System](#4-the-entity-system)
5. [The Game Loop](#5-the-game-loop)
6. [The Event System](#6-the-event-system)
7. [The Script System](#7-the-script-system)
8. [Physics](#8-physics)
9. [AI and Pathfinding](#9-ai-and-pathfinding)
10. [Weapons](#10-weapons)
11. [Player](#11-player)
12. [Rendering](#12-rendering)
13. [Sound](#13-sound)
14. [Networking](#14-networking)
15. [Key C++ vs Java Differences](#15-key-c-vs-java-differences)

---

## 1. Repository Layout

```
neo/
  d3xp/          ← Game logic (entities, AI, weapons, player, scripting)
    ai/           ← AI behaviour and AAS pathfinding
    anim/         ← MD5 skeletal animation
    gamesys/      ← Class/event/save system (the engine's reflection layer)
    menus/        ← In-game UI/shell menus
    physics/      ← Physics backends (player, monster, rigid-body, AF)
    script/       ← Script compiler, interpreter, thread scheduler
  framework/     ← Engine shell: file system, CVars, console, session
  idlib/         ← Math, containers, strings (like java.util + java.lang)
  renderer/      ← OpenGL rendering backend
  sound/         ← Sound system
  cm/            ← Collision model (static geometry)
  aas/           ← Area Awareness System (nav-mesh baking/loading)
  swf/           ← SWF/Flash-based UI renderer
  sys/           ← Platform-specific code (Windows, etc.)
base/
  script/        ← idScript source files (.script) run at game start
doomclassic/     ← Embedded classic DOOM 1/2 emulator
```

---

## 2. Architecture Overview

### Engine / Game split

The engine and game code are separated by a pure-virtual C interface defined in
`neo/d3xp/Game.h`. Think of it as a Java `interface` the game DLL must implement:

```
// Analogous to: public interface IGame { void init(); void runFrame(...); ... }
class idGame {               // neo/d3xp/Game.h:64
    virtual void Init() = 0;
    virtual void RunFrame(idUserCmdMgr&, gameReturn_t&) = 0;
    virtual bool Draw(int clientNum) = 0;
    // ...
};
```

The concrete implementation is `idGameLocal` (`neo/d3xp/Game_local.h:242`).

At startup, the engine calls the exported C function `GetGameAPI()`, which hands back
pointers to `idGame` and `idGameEdit`. This is the only link between engine and game —
like dependency injection through a service locator.

### Subsystem pointers injected into the game

When `GetGameAPI` is called, the engine passes a `gameImport_t` struct containing
interfaces to all engine services the game needs (`neo/d3xp/Game.h:313`):

| Field | Java analogy |
|---|---|
| `idRenderSystem* renderSystem` | `RenderService` singleton |
| `idSoundSystem* soundSystem` | `SoundService` singleton |
| `idFileSystem* fileSystem` | `java.nio.file` |
| `idCVarSystem* cvarSystem` | `java.util.prefs.Preferences` |
| `idCmdSystem* cmdSystem` | Command-pattern dispatcher |
| `idCollisionModelManager*` | Broad-phase collision service |
| `idDeclManager*` | Asset/declaration registry |

---

## 3. The Type / Reflection System

DOOM 3 implements its own runtime type information (RTTI) and reflection layer on top
of C++, because the built-in C++ RTTI was too slow and lacked features the game
needed (factory construction, save/restore, event dispatch).

### idClass — the root object

Every game object ultimately inherits from `idClass` (`neo/d3xp/gamesys/Class.h`).
Think of it as Java's `Object`, except it also carries a static `idTypeInfo` that acts
like a `Class<?>` object.

### Registering a new class — the macros

Two macros replace Java annotations + `@RegisterBean`-style wiring:

```cpp
// In the .h file — like declaring implements Serializable + factory marker
CLASS_PROTOTYPE( idWeapon );       // neo/d3xp/gamesys/Class.h:92

// In the .cpp file — wires up the type chain, event table, spawn/save/restore
CLASS_DECLARATION( idAnimatedEntity, idWeapon )  // Class.h:110
    EVENT( EV_Weapon_State, idWeapon::Event_WeaponState )
    // ...
END_CLASS
```

`CLASS_DECLARATION` generates:
- a static `idTypeInfo` with the class name, superclass name, and a factory function
  (`CreateInstance`) that calls `new idWeapon`.
- An event callback table (discussed in §6).

`idTypeInfo` stores the entire class hierarchy as a linked list, so
`IsType( idActor::Type )` walks up the chain exactly like `instanceof` in Java.

### Spawn lifecycle

When a map entity is instantiated, the engine:
1. Looks up `"classname"` in the entity's key/value dictionary (`idDict spawnArgs`).
2. Finds the matching `idTypeInfo` by name (like `Class.forName()`).
3. Calls `CreateInstance()` (the factory — like `Class.newInstance()`).
4. Calls `Spawn()` on the new object (like a post-construct `@PostConstruct` method).

---

## 4. The Entity System

### Class hierarchy

```
idClass
└── idPhysics          (abstract physics object)
└── idEntity           (neo/d3xp/Entity.h:163) — base game object
    └── idAnimatedEntity  (Entity.h:607) — adds MD5 skeletal animation
        └── idAFEntity_Base   (AFEntity.h:146) — adds articulated-figure ragdoll
            └── idAFEntity_Gibbable (AFEntity.h:214) — can be gibbed/exploded
                └── idActor   (Actor.h:109) — living being (health, damage, animations)
                    ├── idPlayer  (Player.h:245) — the human player
                    └── idAI      (ai/AI.h:256) — NPC/monster
        └── idWeapon  (Weapon.h:83) — weapon entity attached to the player
    └── idProjectile  (Projectile.h) — rockets, plasma bolts, etc.
```

In Java terms `idEntity` is your abstract base `Entity` class, `idActor` is
`LivingEntity`, `idAI` is `MobEntity`, and `idPlayer` is `PlayerEntity`.

### idEntity key members (`neo/d3xp/Entity.h:163`)

| Member | Purpose |
|---|---|
| `int entityNumber` | Unique index in `gameLocal.entities[]` — like a primary key |
| `idDict spawnArgs` | Key/value property bag set from the map file — like a `Map<String,String>` |
| `idScriptObject scriptObject` | Attached script state and functions |
| `int thinkFlags` | Bitmask controlling which per-frame callbacks run (TH_THINK, TH_PHYSICS, TH_ANIMATE, …) |
| `int health` | Current hit points |
| `idList<idEntityPtr<idEntity>> targets` | Other entities to activate when this one fires |
| `entityFlags_s fl` | Boolean flags packed into a bitfield struct |

### Entity pointer safety — `idEntityPtr<T>`

Raw C++ pointers are not safe across save/loads or entity removals. The engine wraps
entity references in `idEntityPtr<T>` (`Game_local.h:193`), which stores a `spawnId`
integer. Dereferencing resolves through the global entity array and checks whether the
stored spawn-id still matches. This is analogous to a Java `WeakReference<T>` that
returns `null` when the referent is gone.

### Think flags — per-frame opt-in

Rather than calling every virtual method on every entity every frame, each entity has a
`thinkFlags` bitmask. An entity calls `BecomeActive(TH_THINK | TH_PHYSICS)` to opt in
and `BecomeInactive(TH_PHYSICS)` to opt out. The game loop only iterates
`activeEntities` — entities with at least one flag set.

```
TH_THINK          = 1   run Think() each frame
TH_PHYSICS        = 2   run physics simulation each frame
TH_ANIMATE        = 4   advance MD5 animation each frame
TH_UPDATEVISUALS  = 8   push new transform to the renderer
TH_UPDATEPARTICLES = 16 update particle emitters
```

### Signals

Entities can register script-thread callbacks for built-in events using typed signal
constants (`Entity.h:76`). This is similar to Java's observer/listener pattern:

```
SIG_TOUCH     // another entity touched this one
SIG_USE       // player used (activated) this entity
SIG_TRIGGER   // entity was triggered by a target chain
SIG_DAMAGE    // entity received damage
SIG_REMOVED   // entity is being removed from the world
SIG_BLOCKED   // physics movement was blocked
SIG_MOVER_POS1/POS2 // door/mover reached a position
```

---

## 5. The Game Loop

Entry point each frame: `idGameLocal::RunFrame` (`neo/d3xp/Game_local.cpp:2256`).

### Frame step-by-step

```
RunFrame(cmdMgr, ret)
 │
 ├─ Advance frame counter and game time (fast + slow timelines for bullet-time)
 ├─ ServerProcessEntityNetworkEventQueue()   — apply incoming net events
 ├─ SetupPlayerPVS()                         — compute potentially-visible sets
 ├─ SortActiveEntityList()                   — pushers first, then team masters
 │
 ├─ for each entity in activeEntities:       — THE MAIN THINK LOOP
 │    RunEntityThink(ent, cmdMgr)
 │      ├─ if player entity → RunAllUserCmdsForPlayer()
 │      │     player->HandleUserCmds()
 │      │     player->Think()
 │      └─ else → ent.Think()              — virtual dispatch
 │
 ├─ RunTimeGroup2()          — "slow-mo" group of entities (helltime powerup)
 ├─ Deactivate entities with cleared thinkFlags
 ├─ ProcessEntityEvents()    — delayed/deferred events (see §6)
 ├─ ServerWriteSnapshot()    — capture state for network clients
 └─ return gameReturn_t      — may contain "map <nextlevel>" session command
```

### Time model

There are two independent game clocks:

| Clock | Purpose |
|---|---|
| `fast` (TIME_GROUP1) | Normal-speed entities — runs at engine frame rate |
| `slow` (TIME_GROUP2) | Slow-motion entities — scaled by `slowmoScale` (0.0–1.0) |

Slow motion ("Helltime" powerup) puts certain entities in `TIME_GROUP2` so they tick
at a fraction of real time while the player runs at full speed.

---

## 6. The Event System

The event system is DOOM 3's inter-object message bus, used both by C++ game code and
by the embedded script language.

### idEventDef — defining an event signature

```cpp
// Declares an event named "damage" that takes (entity, float) and returns void
const idEventDef EV_Damage( "damage", "ef" );
//                                           ^ format: e=entity, f=float
```

Format characters: `d`=int, `f`=float, `v`=vector, `s`=string, `e`=entity,
`E`=nullable entity, `t`=trace result.

### Sending an event

```cpp
// Post immediately (processed this frame, in event queue order)
ent->ProcessEvent( &EV_Activate, activator );

// Schedule for future delivery (like a Java ScheduledExecutorService)
ent->PostEventSec( &EV_Remove, 5.0f );   // remove in 5 seconds
ent->PostEventMS( &EV_Hide, 500 );       // hide in 500 ms
```

Events posted with a delay are stored in a per-entity linked list and delivered in
future frames — this replaces timer threads.

### Receiving an event — the callback table

Each class declares its handled events in its `CLASS_DECLARATION` block:

```cpp
CLASS_DECLARATION( idEntity, idDoor )
    EVENT( EV_Activate,   idDoor::Event_Activate )
    EVENT( EV_Touch,      idDoor::Event_Touch )
END_CLASS
```

When an event fires, the engine looks up the most-derived class's callback table
(walking up the inheritance chain) and calls the matching handler — this is
essentially a vtable-backed multimethod dispatch, similar to `instanceof` chains but
data-driven and scriptable.

---

## 7. The Script System

DOOM 3 has a custom compiled scripting language (`neo/d3xp/script/`) with its own:

- **Compiler** (`Script_Compiler.cpp`) — parses `.script` files into bytecode
- **Program** (`idProgram` in `Script_Program.h`) — holds the compiled bytecode, type
  definitions, and global variable space (up to 296 KB)
- **Interpreter** (`Script_Interpreter.cpp`) — stack-machine bytecode executor
- **Threads** (`idThread` in `Script_Thread.h`) — cooperative coroutines; multiple
  script threads run per frame and can `yield`, `wait`, or block on signals

In Java terms, think of each `idThread` as a `Fiber`/virtual thread that cooperatively
yields to the scheduler each frame. There is no preemption.

### Entry points

The engine loads two default scripts at map start (`Game.h:41`):

```
script/doom_defs.script    ← type/event declarations (like import statements)
script/doom_main.script    ← calls doom_main() which is the top-level map script
```

Script functions are called `function_t` objects and stored in `idProgram`. Each entity
holds an `idScriptObject` (a block of the program's variable space) that maps to its
script class instance.

### C++ ↔ Script bridge

Script functions call C++ via `idEventDef` — the same event system used between C++
objects. A script thread posts an event to an entity; the engine dispatches it through
the C++ callback table. This makes C++ methods scriptable without any extra glue code.

---

## 8. Physics

Every entity has a pluggable `idPhysics*` pointer (the Strategy pattern). Swapping the
physics object changes how the entity moves and collides.

### Physics implementations (`neo/d3xp/physics/`)

| Class | Used by | Behaviour |
|---|---|---|
| `idPhysics_Static` | World geometry, doors at rest | Immovable, zero CPU |
| `idPhysics_Parametric` | Movers, platforms | Scripted keyframe motion |
| `idPhysics_Player` | `idPlayer` | Custom FPS movement (ground, water, air, noclip) |
| `idPhysics_Monster` | `idAI` | Simplified, gravity, no angular dynamics |
| `idPhysics_RigidBody` | Thrown objects, barrels | Full 6-DOF rigid-body simulation |
| `idPhysics_AF` | Ragdolls, articulated figures | Multi-body articulated figure with constraints |

### Clip system — `idClip` (`neo/d3xp/physics/Clip.h`)

`gameLocal.clip` is the central collision dispatcher. It manages `idClipModel` objects
(convex hulls / bounding volumes) and provides:

```cpp
// Cast a ray; like Physics.raycast in Unity
clip.TracePoint( trace, start, end, contentMask, passEntity );

// Move a clip model; detects first contact along the path
clip.Translation( trace, start, end, clipModel, axis, contentMask, passEntity );

// Overlap test — is anything touching this volume?
clip.Contacts( contacts, maxContacts, origin, bounds, contentMask );
```

Content masks are bit flags (`CONTENTS_SOLID`, `CONTENTS_BODY`, `CONTENTS_PROJECTILE`,
etc.) allowing selective filtering — just like layer masks in other engines.

### PVS — Potentially Visible Set (`neo/d3xp/Pvs.h`)

The world is divided into convex portal-connected areas. Each frame,
`SetupPlayerPVS()` floods out from the player's area through open portals to build a
bitset of visible areas. Entities in non-visible areas skip `Think()` and go
*dormant* — this is the main culling mechanism for CPU cost.

---

## 9. AI and Pathfinding

### AAS — Area Awareness System (`neo/aas/`, `neo/d3xp/ai/AAS.h`)

AAS is the nav-mesh equivalent. It pre-computes a graph of convex walkable areas from
the static BSP geometry. At runtime, `idAI` uses it for:

- `FindPath()` — A\* search returning a list of area transitions
- `FindCover()` — finds an area not in the enemy's line of sight
- `FindAttackPosition()` — finds a spot from which the enemy has LOS to the player

### `idAI` state machine (`neo/d3xp/ai/AI.h:256`)

Monster behaviour is driven by script threads running state functions. Each state
function is a script function that loops until it calls `setState()` to transfer to
another state. Common states are `idle`, `combat`, `dead`. The state machine
transitions look like:

```
idle → (notices player via sight/sound) → combat → (lost player) → search → ...
```

### Move commands (`ai/AI.h:71`)

An `idAI` sets a move goal by calling one of these commands:

```cpp
MOVE_NONE                // stand still
MOVE_FACE_ENEMY          // rotate to face enemy
MOVE_TO_ENEMY            // pathfind toward enemy
MOVE_TO_ATTACK_POSITION  // find and move to an attack position
MOVE_TO_COVER            // pathfind to cover
MOVE_WANDER              // random wandering
```

Each frame, the AI's `Physics_Monster` object steps toward the current waypoint,
checking for obstacles. Attack is triggered via script events that fire projectiles or
melee traces.

### Detection — sight and hearing

```
AI_HEARING_RANGE = 2048 units   // sounds propagate up to this distance
```

Sight uses a ray cast from the monster's eyes to the player. Hearing is triggered when
the player fires a weapon or moves fast; noise events are broadcast as `idSoundShader`
emitters with a radius.

---

## 10. Weapons

### State machine (`neo/d3xp/Weapon.h:44`)

```cpp
enum weaponStatus_t {
    WP_READY,       // idle, ready to fire
    WP_OUTOFAMMO,   // cannot fire
    WP_RELOAD,      // reloading animation playing
    WP_HOLSTERED,   // weapon is put away
    WP_RISING,      // weapon is being raised (just selected)
    WP_LOWERING     // weapon is being put away
};
```

Each weapon is a script-driven `idWeapon` entity (attached to but separate from the
player). The weapon script (`base/script/weapon_*.script`) runs state functions for
firing, reloading, raising, and lowering. C++ calls back into script via events like
`EV_Weapon_State`.

### Ammo

Ammo is tracked in the player's inventory, not in the weapon. There are 16 ammo type
slots (`AMMO_NUMTYPES = 16`). Each weapon declares which slot it consumes. Up to 32
weapons can be registered (`MAX_WEAPONS = 32`).

### Firing

Weapons fire either by:
- **Hitscan**: `gameLocal.clip.TracePoint()` — instant ray cast, result is a `trace_t`
  with the hit surface, entity, and normal.
- **Projectile**: spawning an `idProjectile` entity with an initial velocity. The
  projectile then thinks each frame, moving and testing for collision.

---

## 11. Player

`idPlayer` (`neo/d3xp/Player.h:245`) extends `idActor` and is the most complex entity.

### Key subsystems inside `idPlayer`

| Member | Purpose |
|---|---|
| `idPhysics_Player physicsObj` | FPS movement controller |
| `idInventory inventory` | Weapons, ammo, keys, PDAs |
| `idWeapon* weapon` | Currently active weapon entity |
| `idPlayerView playerView` | View bob, damage kicks, special effects |
| `idAnimState headAnim/torsoAnim/legsAnim` | Three independent animation channels |
| `int health` (from idEntity) | Current HP; death at 0 |
| `int stamina` | Sprint resource; affects heartrate HUD |

### Powerups (`Player.h:99`)

```cpp
BERSERK         // melee damage multiplier
INVISIBILITY    // reduces monster detection range
MEGAHEALTH      // health regen above 100
ADRENALINE      // speed boost
INVULNERABILITY // no damage taken
HELLTIME        // slow-motion (puts player in TIME_GROUP1 while enemies in TIME_GROUP2)
ENVIROSUIT      // radiation protection
```

### Heartrate HUD

The heartrate value drives the heartbeat sound volume. Constants (`Player.h:65`):

```
BASE_HEARTRATE        = 70 bpm   (normal)
ZEROSTAMINA_HEARTRATE = 115 bpm  (exhausted)
MAX_HEARTRATE         = 130 bpm  (maximum adrenaline/damage)
DEAD_HEARTRATE        = 0        (dead)
```

### Input → movement pipeline

```
OS input event
 → idUserCmd (button bits + look angles + move axes)  -- produced by framework
 → cmdMgr.GetUserCmdForPlayer()                        -- queued per player
 → idPlayer::HandleUserCmds()                          -- applied to physics
 → idPhysics_Player::Think()                           -- integrates motion
 → entity origin/axis updated
 → renderer notified via UpdateVisuals()
```

---

## 12. Rendering

The game never calls OpenGL directly. It submits data structures to `idRenderWorld`
and `idRenderSystem`, which batch and sort them for the GPU.

### Key data structures

| Struct | Analogous Java class | Purpose |
|---|---|---|
| `renderEntity_t` | `MeshInstance` | Model handle + transform + shader params |
| `renderLight_t` | `LightComponent` | Dynamic light with colour, radius, projection |
| `renderView_t` | `Camera` | Eye position/orientation, FOV, time |
| `idRenderWorld` | `Scene` | Holds all render entities and lights |

### Entity → renderer flow

Each frame, `idEntity::Present()` is called by `Think()`. It pushes the entity's
current transform and appearance into `renderEntity_t` and calls
`gameRenderWorld->UpdateEntityDef(handle, &renderEntity)`. The renderer then handles
visibility, LOD, and draw-call generation.

### Stencil shadows

DOOM 3 is famous for its per-object stencil shadow volumes. Every dynamic light
generates a shadow volume for every shadow-casting model it can reach. The engine uses
a depth-fail ("Carmack's Reverse") algorithm — though that specific code path is
excluded from this GPL release for patent reasons.

---

## 13. Sound

Every entity that emits sound holds a `refSound_t` (`Game.h:196`) containing:

- `idSoundEmitter*` — a handle to the spatial audio engine
- `const idSoundShader*` — the sound asset definition (like a sound bank entry)
- `soundShaderParms_t` — volume, min/max distance, flags

Calling `idEntity::StartSound("snd_death", SND_CHANNEL_VOICE, ...)` looks up the
key `snd_death` in `spawnArgs`, finds the referenced `idSoundShader` declaration, and
asks the sound system to play it spatially at the entity's position.

---

## 14. Networking

DOOM 3 BFG uses a **client-side prediction + server snapshot** model.

### Snapshots

Every frame the server calls `ServerWriteSnapshot()`, which serialises the state of
all network-synchronised entities (`fl.networkSync == true`) into an `idSnapShot`
binary blob. Clients receive snapshots and call `ClientReadSnapshot()` to apply them.

### Prediction

Between snapshots, clients run `ClientRunFrame()` — the same `Think()` loop as the
server, but using locally predicted user commands. When the next server snapshot
arrives, the client corrects any misprediction. `idNetEvent<N>` (`Entity.h:125`) is a
small helper for detecting boolean state changes across dropped snapshots — it uses an
incrementing counter modulo N rather than a raw bool so that skipped snapshots do not
lose events.

### Reliable messages

Non-positional events (chat, weapon drops, achievement unlocks) are sent as reliable
ordered messages (`GAME_RELIABLE_MESSAGE_*` enum, `Game_local.h:124`) rather than in
snapshots.

---

## 15. Key C++ vs Java Differences

| C++ concept | Java equivalent | Notes |
|---|---|---|
| `idClass*` raw pointer | object reference | No GC; lifetime is manual. Entity removal calls `delete`. |
| `idEntityPtr<T>` | `WeakReference<T>` | Safe cross-frame entity reference; resolves to null if entity is gone |
| `CLASS_PROTOTYPE` / `CLASS_DECLARATION` | `@Component` + factory registration | Generates RTTI and event table at static init time |
| `idDict spawnArgs` | `Map<String, String>` | Used to configure entities from map data |
| `idEventDef` + `PostEvent` | Spring `ApplicationEvent` / message queue | Supports deferred delivery by time |
| `idThread` (script) | `Fiber` / virtual thread | Cooperative coroutine, yields each frame |
| `idCVar` | `java.util.prefs.Preferences` | Runtime-tunable console variables |
| `idList<T>` | `ArrayList<T>` | Custom array list with explicit allocator tags |
| `idHashIndex` | `HashMap` (index only) | Parallel int-array hash map for name→index lookups |
| `thinkFlags` bitmask | `Set<ThinkFlag>` | Per-entity opt-in to per-frame callbacks |
| `TH_PHYSICS` flag | calling `physicsObject.update()` | Physics runs only when this flag is set |
| `idPhysics*` pointer | Strategy interface | Swappable physics backend per entity |
| Static global `gameLocal` | Singleton `GameContext` | Central game state; accessible everywhere |
| `#define MAX_GENTITIES 4096` | constant | Hard upper limit on simultaneously live entities |

### Memory model

There is no garbage collector. Object lifetimes are managed by:

1. **Manual `delete`** — `idEntity::Remove()` queues deletion for end-of-frame.
2. **Block allocators** — `idBlockAlloc<T, N>` pre-allocates slabs to avoid heap
   fragmentation for hot objects like events and particles.
3. **Stack** — short-lived objects (traces, vectors) live on the C call stack, not the
   heap.

Java developers should note that reading a dangling `idEntity*` is undefined behaviour
(crash or data corruption). Always use `idEntityPtr<T>` for persistent cross-frame
references.

### No interfaces, no `instanceof` — use `IsType`

```cpp
// C++ DOOM 3 style
if ( ent->IsType( idAI::Type ) ) {
    idAI *ai = static_cast<idAI*>(ent);
}

// Java equivalent
if (ent instanceof AI ai) { ... }
```

`IsType` walks the `idTypeInfo` chain, which is O(depth of hierarchy) — similar cost
to Java's `instanceof`.

---

## Quick Reference: Where to Start Reading

| Goal | File |
|---|---|
| Understand the game loop | `neo/d3xp/Game_local.cpp:2256` (`RunFrame`) |
| Understand entity lifecycle | `neo/d3xp/Entity.h:163` + `neo/d3xp/Entity.cpp` |
| Understand class/event wiring | `neo/d3xp/gamesys/Class.h` + `gamesys/Event.h` |
| Understand player movement | `neo/d3xp/physics/Physics_Player.cpp` |
| Understand monster AI | `neo/d3xp/ai/AI.h` + `ai/AI.cpp` |
| Understand weapon scripting | `neo/d3xp/Weapon.h` + `base/script/weapon_*.script` |
| Understand the scripting VM | `neo/d3xp/script/Script_Interpreter.cpp` |
| Understand networking | `neo/d3xp/Game_network.cpp` |
