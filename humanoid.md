# Roblox Humanoid System Docs

Humanoid system — how it works, what all the types and properties actually mean, and how to replicate it by writing to offsets yourself
---

## What this covers

- humanoid state machine and all state types
- movement system (walkDirection, luaMoveDirection, walkToPoint)
- health and damage
- appendage/body part cache (fast vs slow accessors)
- rig types (R6 vs R15)
- joint layout and hardcoded attachment offsets
- enums (NameOcclusion, DisplayDistanceType, RigType)
- default values from the constructor
- how to replicate by writing each property to its own offset

---

## Humanoid State Machine

the humanoid runs a full state machine internally. current state is stored as a `HumanoidState` object, not a plain enum — you read the state type from it. these are all the possible states:

| State | What it means |
|-------|---------------|
| `FALLING_DWN` | falling and tumbling |
| `RUNNING` | walking/running normally |
| `RUNNING_NO_PHYS` | running but physics disabled |
| `CLIMBING` | on a ladder |
| `STRAFING_NO_PHYS` | strafing without physics |
| `RAGDOLL` | ragdolled |
| `GETTING_UP` | recovering from ragdoll |
| `JUMPING` | mid-jump |
| `LANDED` | just hit the ground |
| `FLYING` | flying |
| `FREE_FALL` | falling normally |
| `SEATED` | sitting in a seat |
| `PLATFORM_STANDING` | standing on a platform (PlatformStand) |
| `DEAD` | health <= 0, joints broken |
| `SWIMMING` | in water |
| `PHYSICS` | fully physics controlled |
| `xx` / `None` | no state / unset |

the default state on spawn is `FREE_FALL`. `getDead()` just checks if `currentState->getStateType() == DEAD`, not the health value directly.

each state transition can be individually enabled/disabled via `SetStateEnabled` — this is how games lock certain states.

---

## Rig Types

| Value | Name | Internal name |
|-------|------|---------------|
| `0` | R6 | `HUMANOID_RIG_TYPE_R6` |
| `1` | R15 | `HUMANOID_RIG_TYPE_R15` |

default is R6. swapping rig type requires `RobloxScript` security permissions internally — it's not something regular scripts can do freely. when you change rig type it flushes and rebuilds the entire appendage cache.

---

## Appendage / Body Part System

the humanoid caches pointers to all body parts in an internal array called `appendageCache`. there are two ways to access them:

**Slow** — searches the character model by name every time, no buffering. safe to call anytime but slower.

**Fast** — reads directly from the cache. only valid if the cache is up to date. much quicker but can return stale data if the character was modified.

the appendage indices:

| Index | R6 part name | R15 part name |
|-------|--------------|----------------|
| `0` TORSO | `HumanoidRootPart` | `HumanoidRootPart` |
| `1` HEAD | `Head` | `Head` |
| `2` RIGHT_ARM | `Right Arm` | `RightArm` |
| `3` LEFT_ARM | `Left Arm` | `LeftArm` |
| `4` RIGHT_LEG | `Right Leg` | `RightLeg` |
| `5` LEFT_LEG | `Left Leg` | `LeftLeg` |
| `6` VISIBLE_TORSO | `Torso` | `UpperTorso` |

note: `TORSO` (index 0) is the `HumanoidRootPart` — the invisible physics root. `VISIBLE_TORSO` (index 6) is the actual visible torso part. `getTorsoSlow()` tries TORSO first and falls back to VISIBLE_TORSO if it's null.

---

## Joint Layout (R6)

these are the hardcoded joint attachment offsets baked into the source:

```cpp
// torso attachments (part0 side)
rightShoulderP  = normalIdToMatrix3(NORM_X),     Vector3( 1,  0.5, 0)
leftShoulderP   = normalIdToMatrix3(NORM_X_NEG), Vector3(-1,  0.5, 0)
rightHipP       = normalIdToMatrix3(NORM_X),     Vector3( 1, -1,   0)
leftHipP        = normalIdToMatrix3(NORM_X_NEG), Vector3(-1, -1,   0)
neckP           = normalIdToMatrix3(NORM_Y),     Vector3( 0,  1,   0)

// limb attachments (part1 side)
rightArmP  = normalIdToMatrix3(NORM_X),     Vector3(-0.5,  0.5, 0)
leftArmP   = normalIdToMatrix3(NORM_X_NEG), Vector3( 0.5,  0.5, 0)
rightLegP  = normalIdToMatrix3(NORM_X),     Vector3( 0.5,  1,   0)
leftLegP   = normalIdToMatrix3(NORM_X_NEG), Vector3(-0.5,  1,   0)
headP      = normalIdToMatrix3(NORM_Y),     Vector3( 0,   -0.5, 0)
```

joints are all `Motor6D` with `maxVelocity = 0.1`. the root joint connecting `HumanoidRootPart` to `Torso` is also a Motor6D named `RootJoint`, with both C0 and C1 set to `(NORM_Y, Vector3(0,0,0))`.

for R15 the joints are built from `RigAttachment` named attachments instead of these hardcoded offsets.

---

## Health System

```cpp
health    = 100.0f   // default
maxHealth = 100.0f   // default
```

health has two setters internally:
- `setHealth` — raw setter, used for XML/replication, no clamping
- `setHealthUi` — clamps to `[0, maxHealth]` before setting, used for Lua

`takeDamage` checks if the torso is inside a ForceField first — if it is, damage is ignored entirely.

```cpp
void takeDamage(float value) {
    if (!ForceField::partInForceField(torso))
        setHealth(getHealth() - value);
}
```

death is triggered by the state machine, not health directly. joints only break on death if `hadNeck && hadHealth` are both true — this prevents newly spawned humanoids with no history from breaking apart.

---

## Movement System

there are three separate movement inputs that can be active at once, with a priority order:

**1. walkDirection** — set by user input (keyboard/controller), stored as a compacted Vector3 where z is in the y slot. takes priority over luaMoveDirection.

**2. luaMoveDirection** — set by `Humanoid:Move()` from Lua. unit vector. gets cancelled if walkDirection is set.

**3. walkToPoint / walkToPart** — click-to-walk target. gets cancelled when luaMoveDirection is set.

```cpp
// desired velocity calculation (simplified):
Vector3 intendedMovement = walkDirection;
if (intendedMovement == zero)
    intendedMovement = luaMoveDirection;

// if click-to-walk is active, override with direction to goal
if (isWalking && canClickToWalk())
    intendedMovement = normalize(getDeltaToGoal());

finalVelocity = walkSpeed * intendedMovement * percentWalkSpeed;
```

`percentWalkSpeed` is clamped to `[0, 1]` and is used for analog input (thumbsticks). it scales the final velocity without changing the actual WalkSpeed value.

`walkAngleError` is used when `AutoRotate` is on — it drives the rotational velocity to turn the character toward its movement direction.

### MoveTo / click-to-walk

`moveTo` sets `walkToPoint` and `walkToPart` then starts the walk timer. the timer is:
- `8.0` seconds (cyclic executive mode)
- `240` ticks (legacy mode)

when the character gets within `1.0` stud of the goal (squared distance < 1), `MoveToFinished(true)` fires. if the timer runs out first, `MoveToFinished(false)` fires.

the arrival check adds `Vector3(0, 3, 0)` to the goal before comparing — that's the height of the legs + half torso so it checks at torso level not floor level.

---

## WalkSpeed and JumpPower clamping

```cpp
// walkSpeed: negative values clamped to 0 (if HumanoidCheckForNegatives flag is on)
// jumpPower: clamped to [0, 1000]
// maxSlopeAngle: clamped to [0, 89.9]
// percentWalkSpeed: clamped to [0, 1]
// nameDisplayDistance / healthDisplayDistance: clamped to >= 0
```

default values from the constructor:

```cpp
walkSpeed       = 16.0f
jumpPower       = kJumpVelocityGrid()  // internal constant
maxSlopeAngle   = 89.0f
hipHeight       = 0.0f
health          = 100.0f
maxHealth       = 100.0f
autorotate      = true
autoJumpEnabled = true
ragdollCriteria = 34
nameDisplayDistance   = 100.0f
healthDisplayDistance = 100.0f
nameOcclusion         = NAME_OCCLUSION_NONE
displayDistanceType   = HUMANOID_DISPLAY_DISTANCE_TYPE_VIEWER
rigType               = HUMANOID_RIG_TYPE_R6
previousState         = FREE_FALL
```

---

## Enums

### NameOcclusion

| Value | Name | What it does |
|-------|------|--------------|
| `0` | None | name always visible |
| `1` | EnemyOccluded | name hidden from enemies |
| `2` | OccludeAll | name always hidden |

### HumanoidDisplayDistanceType

| Value | Name | What it does |
|-------|------|--------------|
| `0` | Viewer | distance measured from the viewer's camera |
| `1` | Subject | distance measured from the humanoid itself |
| `2` | None | never displayed |

---

## Replicating it yourself (writing to offsets)

same deal as the other docs — no single write, every property goes to its own offset individually.


### setup

```cpp
#include <Windows.h>
#include <cmath>

namespace Offsets {
    uintptr_t HumanoidBase        = 0x????????;

    // floats
    uintptr_t WalkSpeed           = 0x???;
    uintptr_t Health              = 0x???;
    uintptr_t MaxHealth           = 0x???;
    uintptr_t JumpPower           = 0x???;
    uintptr_t HipHeight           = 0x???;
    uintptr_t MaxSlopeAngle       = 0x???;
    uintptr_t WalkAngleError      = 0x???;
    uintptr_t PercentWalkSpeed    = 0x???;

    // Vector3s
    uintptr_t WalkDirection       = 0x???;  // compacted, z stored in y slot
    uintptr_t LuaMoveDirection    = 0x???;  // unit vector
    uintptr_t WalkToPoint         = 0x???;
    uintptr_t TargetPoint         = 0x???;
    uintptr_t CameraOffset        = 0x???;

    // bools
    uintptr_t Jump                = 0x???;
    uintptr_t Sit                 = 0x???;
    uintptr_t AutoRotate          = 0x???;
    uintptr_t AutoJumpEnabled     = 0x???;
    uintptr_t PlatformStanding    = 0x???;
    uintptr_t Strafe              = 0x???;
    uintptr_t HeadLocked          = 0x???;

    // ints / enums
    uintptr_t RigType             = 0x???;  // 0 = R6, 1 = R15
    uintptr_t NameOcclusion       = 0x???;  // 0/1/2
    uintptr_t DisplayDistanceType = 0x???;  // 0/1/2
    uintptr_t RagdollCriteria     = 0x???;

    // state (read only — written by the state machine)
    uintptr_t CurrentStateType    = 0x???;  // int, see state enum above
}

HANDLE hProcess = nullptr;
```

### writing properties

floats and ints write directly, Vector3s write as a 12-byte block:

```cpp
// float
float speed = 50.0f;
WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::WalkSpeed), &speed, sizeof(float), nullptr);

// bool
bool jump = true;
WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Jump), &jump, sizeof(bool), nullptr);

// Vector3 — 3 floats contiguous, write as one block
float dir[3] = { 0.0f, 0.0f, -1.0f }; // moving forward
WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LuaMoveDirection), dir, sizeof(float) * 3, nullptr);

// int / enum
int rigType = 1; // R15
WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::RigType), &rigType, sizeof(int), nullptr);
```

### reading health and state

```cpp
float ReadHealth(uintptr_t base) {
    float val = 0.0f;
    ReadProcessMemory(hProcess, (LPCVOID)(base + Offsets::Health), &val, sizeof(float), nullptr);
    return val;
}

int ReadStateType(uintptr_t base) {
    int val = 0;
    ReadProcessMemory(hProcess, (LPCVOID)(base + Offsets::CurrentStateType), &val, sizeof(int), nullptr);
    return val;
}

bool IsDead(uintptr_t base) {
    return ReadStateType(base) == 13; // DEAD = 13 in the enum order
}
```

### quick reference — things worth writing to

| Property | Type | Notes |
|----------|------|-------|
| `WalkSpeed` | float | default 16, negative clamped to 0 |
| `Health` | float | setting below 0 triggers death state |
| `MaxHealth` | float | affects health bar display |
| `JumpPower` | float | clamped to [0, 1000] |
| `Jump` | bool | set true to trigger a jump |
| `Sit` | bool | force sit/unsit |
| `AutoRotate` | bool | disabling stops character from auto-turning |
| `PlatformStand` | bool | disables normal movement |
| `LuaMoveDirection` | Vector3 | unit vector, moves humanoid continuously |
| `CameraOffset` | Vector3 | offsets camera target from humanoid |
| `NameOcclusion` | int | 0/1/2, controls nametag visibility |

---

*more systems coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
