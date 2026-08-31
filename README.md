# Multiplayer-Party-Game
The open-source version of this project will be released here 6 months after its Steam launch.

## Update
Playtest video from Deathrun game mode: https://www.youtube.com/watch?v=m6DaPlQOnzM


# 🎮 Unity Multiplayer Game

> [!NOTE]
> All scripts, architectural decisions, and game modes are explained in detail.

---

## Overview

An online multiplayer FPS game developed with Unity + **Mirror Networking** + **Steamworks.NET** (FizzySteamworks transport), supporting multiple game modes.

Devlog videos: https://www.youtube.com/watch?v=VipbIQdGEzM&list=PLf6y277d--h0

| Feature | Details |
|---|---|
| **Networking** | Mirror (NetworkBehaviour, SyncVar, Command, ClientRpc) |
| **Platform** | Steam (Steamworks.NET, FizzySteamworks transport) |
| **Render Pipeline** | URP (Universal Render Pipeline) |
| **Physics** | Rigidbody-based (Interpolation, Zero Friction PhysicMaterial) |
| **Input** | Legacy Input System (Input.GetAxis, Input.GetKey) |

---

## 📁 Project Structure

```
Assets/Scripts/
├── PlayerController/                ← Modular Partial Class system
│   ├── PlayerController.cs          ← Main file (variables, Start, Update flow, LateUpdate)
│   ├── PlayerController.Movement.cs ← Movement physics (Bhop, Surf, Ice, Platform)
│   ├── PlayerController.Weapon.cs   ← Weapon system (Equip, Shoot, Reload, ADS, Recoil, Bob/Sway)
│   ├── PlayerController.Network.cs  ← Network sync (Heartbeat, Teleport, SpeedRun Finish, Reset)
│   ├── PlayerController.Interaction.cs ← E key trap button interaction
│   └── PlayerController.Audio.cs    ← Surface-based footstep sounds
│
├── Network/
│   ├── MyNetworkManager.cs          ← Custom NetworkManager (DontDestroy, music, disconnect)
│   └── SteamLobby.cs               ← Lobby creation, joining, mode/map selection, code system
│
├── Deathrun Scripts/
│   ├── DeathrunManager.cs           ← State Machine (Waiting → Voting → Starting → Playing → Ended)
│   ├── DeathrunUIManager.cs         ← Timer, voting panel, winner screen, interact prompt
│   └── CinematicSpaceShip.cs        ← Cinematic spaceship roaming the map
│
├── SpeedRun/
│   ├── SpeedRunManager.cs           ← Map timer, best times (SyncDictionary), finish logging
│   ├── SpeedRunUIManager.cs         ← HUD (run time, remaining time, TAB scoreboard)
│   └── SpeedRunFinishLine.cs        ← Finish line trigger
│
├── Trap/
│   ├── DeathrunTrapBase.cs          ← Abstract trap base (sound, RPC activation)
│   ├── DeathrunTrapButton.cs        ← E key trap trigger button (cooldown, killerOnly)
│   ├── Trap_Move.cs                 ← Moving platform trap
│   ├── Trap_Rotate.cs              ← Rotating obstacle (tangential velocity calculation)
│   ├── Trap_Fan.cs                  ← Wind-blowing fan (VFX, PlayerController.AddWindForce)
│   ├── Trap_FireBridge.cs           ← Fire bridge (DPS damage, SyncVar particles)
│   ├── Trap_GlassPath.cs           ← Glass path (breakable panels)
│   ├── Trap_KillersDeath.cs         ← Final trap — Runners kill all Killers
│   ├── DeadlyObstacle.cs            ← Instant-kill obstacle on contact (Trigger/Collision)
│   ├── DeathZone.cs                 ← Map fall-off zone
│   └── IceFloor.cs                  ← Ice floor (acceleration, max speed, steering force)
│
├── UI Scripts/
│   ├── PlayerHUD.cs                 ← Health, ammo, weapon name, hitmarker, damage overlay, connection
│   ├── ScoreboardManager.cs         ← TAB scoreboard (Deathrun-specific Runner/Killer separation)
│   ├── ScoreboardItem.cs            ← Scoreboard row prefab
│   ├── InGameMenu.cs                ← ESC pause menu (Disconnect, Quit)
│   ├── LobbyItem.cs                 ← Lobby list row prefab
│   ├── MainMenuManager.cs           ← Main menu host/join bridge
│   └── Crosshair.cs                 ← Pixel-based crosshair (OnGUI)
│
├── Steamworks.NET/
│   └── SteamManager.cs              ← Steam API initialization (Valve standard code)
│
├── Health.cs                        ← Health system (TakeDamage, Die, Respawn, Spectate mode)
├── WeaponData.cs                    ← ScriptableObject weapon definition (full loadout)
├── ArenaMirror.cs                   ← 360° real-time cubemap mirror
├── Portal_Controller.cs             ← Portal teleportation system (momentum preservation)
└── CameraController.cs              ← Simple follow camera (disabled when PlayerController takes over)
```

---

## 🕹️ Game Modes

### 1. Deathrun
- **Teams:** `Runner` vs `Killer` (`Team` enum)
- **State Machine** (DeathrunManager):
  ```
  WaitingForPlayers → KillerSelectionVoting → RoundStarting (5s freeze) → RoundPlaying → RoundEnded
  ```
- Every 3 rounds: `KillerOptOutVoting` (Do you want to stay as Killer?)
- 6+ players = 2 Killers, <6 = 1 Killer
- **Runner**: Knife only (slot 2), weapon switching disabled
- **Killer**: All weapons, can press trap buttons (`killerOnly`)
- **Win conditions**: All Runners dead → Killers Win, all Killers dead → Runners Win, time runs out → Killers Win
- **Round end**: Slow motion (timeScale=0.3), victory/defeat sound, gradient text
- **Trap reset**: `DeathrunManager.OnRoundReset` global event resets all traps and buttons
- **Death**: Transition to Spectator mode (teleport to waiting room, invisible but can move)

### 2. Speedrun
- **Solo race** (multiple players, collision disabled = ghosting)
- **CS:GO-style Bhop**: Auto-bhop (hold Space), air-strafing, momentum preservation
- **Surf mechanics**: Sliding on surfaces angled 40°–88°, gaining speed
- **Ice floor**: IceFloor component, acceleration + max speed + steering
- **Knife only**, weapon switching and dealing damage disabled
- **Finish line** (SpeedRunFinishLine): Trigger → CmdReportSpeedRunFinish → 3s freeze → respawn
- **Best times**: SyncDictionary<netId, float>, displayed on TAB scoreboard
- **Map timer**: 10 minutes default, triggers EndSpeedRunMode when expired

### 3. Other Modes (Future)
- Infrastructure ready for "Deathmatch", "1v1", etc. (gameModes list, map dropdown)

---

## 👤 PlayerController — Modular Partial Class Architecture

### Main File (`PlayerController.cs`)
- `NetworkBehaviour`, all variable declarations
- `Team` enum: None, Runner, Killer
- SyncVars: `playerName`, `currentTeam`, `score`, `kills`, `deaths`, `isSpectating`, `currentWeaponIndex`, `syncPitch`, `speedRunStartTime`
- `Start()`: NetworkAnimator, AudioSource init
- `OnStartClient()`: Speedrun ghosting (Physics.IgnoreCollision)
- `OnStartLocalPlayer()`: Steam name, cursor lock, Rigidbody, camera parenting, ZeroFriction material, FPS model hiding (ShadowsOnly), weapon equip
- `Update()`: Footstep → TPS recoil recovery → Heartbeat → LocalPlayer controls → Mouse rotation → Movement input → Ground check → Weapon switching → Interaction → Animator → Combat → Camera → Physics movement
- `LateUpdate()`: Spine bone pitch sync, TPS recoil, flinch

### Movement (`PlayerController.Movement.cs`)
- `HandleMovement()`: Platform velocity (Trap_Move/Trap_Rotate), sprinting, jumping, mode branching → `CalculateSpeedrunMovement()` or `CalculateDeathrunMovement()`
- **Speedrun physics**: Ground friction, air-strafing (100f accel), surf mechanics, ice momentum, speed caps (walk×5, surf×8)
- **Deathrun physics**: Standard walk/run, weapon speed multiplier, burn slowdown, ice sliding
- `AddWindForce()`: Wind from Fan trap
- `OnCollisionStay/Exit`: Surf surface detection (angle 40°–88°), wall-stick prevention

### Weapon (`PlayerController.Weapon.cs`)
- 3-slot weapon system: `equippedWeapons[3]`
- FPS + TPS model spawning (under WeaponCamera + on RightHand bone)
- ADS: FOV transition + position interpolation
- Procedural reload: Weapon lower/raise animation, sequential sounds
- Recoil: Position + rotation kickback (snappiness/returnSpeed)
- Melee: Light (Fire1) and heavy (Fire2) attacks, separate damage/speed
- MuzzleFlash: ParticleSystem + Light flash (0.04s)
- Network: CmdShoot → RpcShowImpact + RpcApplyTPSRecoil + RpcPlayGunshot
- Damage: CmdDealDamage (friendly fire prevention, mode checks)
- Weapon switching rules: Disabled in Speedrun, Deathrun Runners knife-only

### Network (`PlayerController.Network.cs`)
- **Heartbeat**: Server→Client RPC, 1s interval, warning at 3s, disconnect at 13s
- **ConnectionInterrupted**: Countdown display on HUD
- **Teleport**: `RpcTeleport(pos, rot)` → Rigidbody reset
- **SpeedRun Finish**: `CmdReportSpeedRunFinish` → freeze → instant respawn
- **ResetPlayerState**: Revive, weapon/ammo reset

### Interaction (`PlayerController.Interaction.cs`)
- `TryInteract()`: Raycast 4m → finds DeathrunTrapButton → `CmdActivateTrap()`
- `UpdateInteractPrompt()`: UI indicator (DeathrunUIManager)

### Audio (`PlayerController.Audio.cs`)
- Surface-based footstep sounds: Different sound arrays based on surface tag
- Interval and volume adjusted based on walk/run speed
- Plays on all players (remote players at 60% volume)

---

## 🔫 Weapon System (WeaponData ScriptableObject)

| Category | Properties |
|---|---|
| **General** | weaponName, weaponPrefab, weaponType (Pistol/Rifle/Melee) |
| **Animation** | FPS/TPS equip/reload trigger names, animatorOverride |
| **Positioning** | FPS/TPS position, rotation, scale |
| **Combat** | damage, fireRate, range, heavyDamage, heavyFireRate |
| **Ammo** | magSize, reserveAmmo, reloadDuration |
| **Movement** | walkSpeedMultiplier, runSpeedMultiplier, animatorSpeedMultiplier |
| **Sway/Bob** | swayAmount, maxSway, bobSpeed/Amount (idle/walk/run), ADS multipliers |
| **ADS** | aimPosition, aimSmooth, adsSmooth, aimFov |
| **Recoil** | recoilX/Y/Z, snappiness, returnSpeed, ADS recoil, TPS recoil |
| **Audio** | gunshotSound, reloadSounds[], equipSound, maxAudioRange |

---

## ❤️ Health System (Health.cs)

- `maxHealth = 100`, SyncVar hook updates HUD + damage overlay
- **Death flow**: `TakeDamage()` → `RpcDie()` (isDead, kinematic, animator bool) → `FreezeDeathAnimationRoutine()` (invisible after 2s)
- **Respawn modes**:
  - **Normal**: 4s delay → random spawn → `RpcRespawn()` → kinematic trick
  - **SpeedRun**: 0.05s instant → start position → reset speedRunStartTime
  - **Deathrun**: 4s → spectator mode (`isSpectating=true`) → teleport to waiting room
- **Safety**: Prevents damage outside RoundPlaying state (prevents bugs during teleportation)

---

## 🌐 Network Architecture

### MyNetworkManager
- `DontDestroyOnLoad` enforcement, singleton protection
- Menu music (loop, stops when client connects, resumes on disconnect)
- `OnClientDisconnect`: Error message + return to main menu
- `ReturnToMenu()`: Calls `SteamLobby.OnDisconnect()`

### SteamLobby
- **Lobby creation**: Name, game mode, map, player count → `SteamMatchmaking.CreateLobby()`
- **Mode & Map system**: `GameModeInfo` struct → each mode has its own map list (`MapInfo`)
- **Static variables**: `currentGameMode`, `currentMapName`, `currentLobbyCode`
- **Lobby code**: 5-character random (A–Z, 0–9), written to Steam lobby metadata
- **Code search**: `SearchAndJoinWithCode()` → string filter → auto-join first match
- **Connection timeout**: Error message + flashing text after 5s
- **GameID filtering**: `"cb2soft_Deathrun"` prevents mixing with other games
- Deathrun mode max 12 players, others max 16

---

## 🗺️ Scenes

| Scene | Mode | Description |
|---|---|---|
| `SampleScene` | Main Menu | Entry screen |
| `DM_Test` | Deathmatch? | Test scene |
| `TEST_SCENE` | Test | General testing |
| `DR_Desert` | Deathrun | Desert themed |
| `DR_Space` | Deathrun | Space themed (large scene, 13MB) |
| `DR_Winter` | Deathrun | Winter themed |
| `SR_Space` | Speedrun | Space themed speedrun |

---

## 🪤 Trap System

All traps inherit from the `DeathrunTrapBase` abstract class. Triggered via `DeathrunTrapButton` with the E key.

| Trap | Features |
|---|---|
| **Trap_Move** | Moving platform (round-trip or permanent) + player carrying (currentVelocity) |
| **Trap_Rotate** | Rotating obstacle + tangential velocity calculation (Cross Product) for player push |
| **Trap_Fan** | VFX wind + propeller spin + player push (AddWindForce) |
| **Trap_FireBridge** | SyncVar fire effect + DPS damage per second + burn slowdown |
| **Trap_GlassPath** | Randomly breaking glass panels (left/right), restored on round reset |
| **Trap_KillersDeath** | Final trap — Runner activates it to deal DPS damage to all Killers + global alarm + +30s time |
| **DeadlyObstacle** | Instant-kill obstacle on contact/trigger (9999 damage) |
| **DeathZone** | Map fall-off zone |
| **IceFloor** | Ice floor parameters (acceleration, max speed, steering) |

All traps subscribe to the `DeathrunManager.OnRoundReset` event and reset themselves at the end of each round.

---

## 🎨 Other Systems

### ArenaMirror
- 360° real-time cubemap rendering
- Custom URP shader (`Custom/CubemapMirror`)
- Configurable resolution, update rate, culling mask

### Portal_Controller
- Teleportation system (portal effect + teleport)
- Momentum preservation: Transfers entry velocity to exit direction
- `PortalTrigger` subclass: Helper that automatically finds the Collider

### CameraController
- Simple third-person camera follow
- Disabled when PlayerController takes over

---

## 📦 Other Folders

| Folder | Contents |
|---|---|
| `Characters/` | Character models (ithappy City_Characters asset) |
| `WeaponLibrary/` | Weapon prefabs and WeaponData assets |
| `Prefabs/` | DR_OBJECTS, PLAYER_PREFABS, Manager Prefabs, UI_PREFABS, WEAPON PREFABS, TEST OBJECTS |
| `Animations/` | Character animations |
| `Audio/` | Sound files |
| `Materials/` | Materials |
| `Shader/` | Custom shaders (CubemapMirror, etc.) |
| `VFX/` | Visual Effect Graph files |
| `Terrains/` | Terrain files |
| `_Recovery/` | Recovery files |

---

## 🔑 Key Architectural Decisions

1. **Partial Class usage**: PlayerController has a 590+ line main file + 5 partial files (~1800 total lines). Makes maintenance easier.
2. **SteamLobby.currentGameMode static string**: Game mode is checked across all systems using this string (`Equals` + OrdinalIgnoreCase).
3. **Rigidbody-based movement**: Sets `rb.linearVelocity` in Update + `rb.AddForce` for jump. FixedUpdate is empty.
4. **ZeroFriction PhysicMaterial**: Prevents sticking to walls.
5. **Heartbeat system**: Server→Client RPC, warning at 3s, disconnect at 13s. Provides early notification of connection loss.
6. **DeathrunManager.OnRoundReset event**: Central event for resetting all traps and buttons at round end.
7. **FPS+TPS weapon model**: Each weapon is spawned separately under WeaponCamera (FPS) and on the RightHand bone (TPS). FPS arm meshes are hidden in TPS.
8. **Spectator mode**: Players who die in Deathrun are teleported to the waiting room, distinguished on the scoreboard via `isSpectating=true`.

---

## ⚠️ Known Issues / Incomplete Work

1. **Character selection**: Started but completion status is unclear
2. **SpeedRun EndSpeedRunMode**: Action when map time expires is still empty (leaderboard, etc.)
3. **Battle Royale mode**: Not yet developed
4. **Surf mechanics**: Working on, appears to be implemented in current code
