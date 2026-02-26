# Roblox Workspace Docs

been reversing the workspace class and figured i'd write it up the same way i did the lighting docs. covers the internal properties, what they do, and how to read/write them.

if this helps you out, drop a ⭐ or come hang: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)

---

## What this covers

- game time — distributedGameTime and how it syncs
- physics — PGS solver, fallen parts, throttling, awake parts
- network flags — streaming, filtering, third party sales
- camera — current camera pointer and how it's managed
- terrain — how the terrain instance is stored and accessed
- physics analyzer — the PGS inconsistency detector
- **how to read/write all of these yourself via offsets**

---

## Game Time

`distributedGameTime` is a double that tracks the current game time in seconds. it's separate from clock time — it's the simulation clock that physics and game logic runs on. on the server it gets set from `RunService::gameTime()`. on the client it's updated but not broadcast back.

```cpp
double getDistributedGameTime(uintptr_t base) {
    double value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::DistributedGameTime), &value, sizeof(double), nullptr);
    return value;
}

void setDistributedGameTime(uintptr_t base, double value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::DistributedGameTime), &value, sizeof(double), nullptr);
}
```

> **note:** setting this directly on the client bypasses the server sync. the game internally has a `setDistributedGameTimeNoTransmit` path that does exactly this — writes the value without firing a property change event. if you want to avoid triggering replication, write directly to the offset.

---

## Physics

### FallenPartsDestroyHeight

a float clamped to `[-50000, 50000]`. parts that fall below this y value get destroyed. defaults to whatever the world's internal value is.

```cpp
float getFallenPartDestroyHeight(uintptr_t base) {
    float value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::FallenPartDestroyHeight), &value, sizeof(float), nullptr);
    return value;
}

void setFallenPartDestroyHeight(uintptr_t base, float value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::FallenPartDestroyHeight), &value, sizeof(float), nullptr);
}
```

### PGSPhysicsSolverEnabled (ExperimentalSolverEnabled)

bool that enables the PGS physics solver. setting this also syncs `expSolverEnabled_Replicate` — the two are kept in lockstep internally. stored in the world's kernel.

```cpp
bool getExperimentalSolverEnabled(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::ExperimentalSolverEnabled), &value, sizeof(bool), nullptr);
    return value;
}

void setExperimentalSolverEnabled(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ExperimentalSolverEnabled), &value, sizeof(bool), nullptr);
}
```

> **note:** there are two separate flags here — `experimentalSolverEnabled` and `expSolverEnabled_Replicate`. internally setting one syncs the other. if you write directly to the offset, only the one you wrote changes. if that matters for your use case, write both.

### ExpSolverEnabled_Replicate

the replication-facing copy of the PGS flag. this is what gets streamed to clients.

```cpp
bool getExpSolverEnabled_Replicate(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::ExpSolverEnabled_Replicate), &value, sizeof(bool), nullptr);
    return value;
}

void setExpSolverEnabled_Replicate(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ExpSolverEnabled_Replicate), &value, sizeof(bool), nullptr);
}
```

### PhysicalPropertiesMode

an enum with three values — `Default (0)`, `Legacy (1)`, `New (2)`. controls whether parts use the new physical material properties system. can only be changed outside of runtime (the setter checks run state and no-ops if the simulation is running or paused).

```cpp
int getPhysicalPropertiesMode(uintptr_t base) {
    int value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::PhysicalPropertiesMode), &value, sizeof(int), nullptr);
    return value;
}

void setPhysicalPropertiesMode(uintptr_t base, int value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::PhysicalPropertiesMode), &value, sizeof(int), nullptr);
}
```

---

## Network Flags

### StreamingEnabled

bool. enables network streaming — data is gradually streamed to the client rather than all at once. server-only, not replicated.

```cpp
bool getNetworkStreamingEnabled(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::NetworkStreamingEnabled), &value, sizeof(bool), nullptr);
    return value;
}

void setNetworkStreamingEnabled(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::NetworkStreamingEnabled), &value, sizeof(bool), nullptr);
}
```

### FilteringEnabled (NetworkFilteringEnabled)

bool. enables network filtering / FE mode. when enabled, client changes to the workspace are not replicated to the server. setting this on the server also triggers a GA event the first time.

```cpp
bool getNetworkFilteringEnabled(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::NetworkFilteringEnabled), &value, sizeof(bool), nullptr);
    return value;
}

void setNetworkFilteringEnabled(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::NetworkFilteringEnabled), &value, sizeof(bool), nullptr);
}
```

### AllowThirdPartySales

bool. controls whether third party game passes can be sold in this place.

```cpp
bool getAllowThirdPartySales(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::AllowThirdPartySales), &value, sizeof(bool), nullptr);
    return value;
}

void setAllowThirdPartySales(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::AllowThirdPartySales), &value, sizeof(bool), nullptr);
}
```

---

## Camera

### CurrentCamera

a pointer to the currently active `Camera` instance. if no camera exists as a descendant, it falls back to `utilityCamera` (an internal camera not shown in the tree). the pointer is a `shared_ptr<Camera>` internally so what you read at the offset is the raw pointer inside it.

```cpp
uintptr_t getCurrentCamera(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::CurrentCamera), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // raw Camera* 
}

void setCurrentCamera(uintptr_t base, uintptr_t cameraPtr) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::CurrentCamera), &cameraPtr, sizeof(uintptr_t), nullptr);
}
```

> **note:** the actual field is a `shared_ptr<Camera>`, so the raw pointer is at `base + Offsets::CurrentCamera`, not offset by the shared_ptr's internal bookkeeping. if your read comes back garbage, try reading `base + Offsets::CurrentCamera + 0x8` (the px field of the shared_ptr on MSVC).

### RenderingDistance

a float controlling the max rendering distance. initialized to `10000.0f`.

```cpp
float getRenderingDistance(uintptr_t base) {
    float value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::RenderingDistance), &value, sizeof(float), nullptr);
    return value;
}

void setRenderingDistance(uintptr_t base, float value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::RenderingDistance), &value, sizeof(float), nullptr);
}
```

---

## Terrain

a pointer to the `MegaClusterInstance` terrain object. it's stored as a `shared_ptr<Instance>` and has its parent locked so it can't be moved. same shared_ptr caveat as CurrentCamera applies.

```cpp
uintptr_t getTerrain(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Terrain), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // raw Instance*
}
```

> **note:** terrain is read-only from an offset perspective — you can't meaningfully "set" it by writing a pointer. the terrain instance locks its own parent on creation and clears itself through `clearTerrain()`. writing a pointer here won't set up those locks.

---

## Debug/Stats Flags

these are static bools on the `Workspace` class — they're class-level, not per-instance. that means they're not at an instance offset, they're at a fixed address in the module.

```
Workspace::showWorldCoordinateFrame  — renders XYZ axis at origin
Workspace::showHashGrid              — renders the spatial hash grid
Workspace::showEPhysicsOwners        — highlights physics ownership regions
Workspace::showEPhysicsRegions       — highlights physics regions
Workspace::showStreamedRegions       — highlights streamed regions
Workspace::showPartMovementPath      — shows movement path for parts
Workspace::showActiveAnimationAsset  — shows active animation asset info
Workspace::gridSizeModifier          — float, controls 3D grid density (default 4.0)
```

all initialized to `false` (except `gridSizeModifier = 4.0f`). since these are statics, read/write them at their module offset directly, not relative to a workspace instance base.

```cpp
// example — read showEPhysicsOwners
bool getShowEPhysicsOwners(uintptr_t moduleBase) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(moduleBase + Offsets::showEPhysicsOwners), &value, sizeof(bool), nullptr);
    return value;
}

void setShowEPhysicsOwners(uintptr_t moduleBase, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(moduleBase + Offsets::showEPhysicsOwners), &value, sizeof(bool), nullptr);
}
```

---

## Physics Analyzer

only active when `PhysicsAnalyzerEnabled` FFlag is on AND PGS solver is enabled. tracks inconsistent body pairs (constraint solver failures) each step.

```cpp
// read the break-on-issue flag (requires PGS to be enabled or always returns false)
bool getPhysicsAnalyzerBreakOnIssue(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::PhysicsAnalyzerBreakOnIssue), &value, sizeof(bool), nullptr);
    return value;
}

void setPhysicsAnalyzerBreakOnIssue(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::PhysicsAnalyzerBreakOnIssue), &value, sizeof(bool), nullptr);
}
```

> **note:** `getPhysicsAnalyzerIssue(int group)` returns a list of `PartInstance*` involved in a constraint inconsistency group. this is exposed to Lua under `Security::Plugin`. not accessible externally through memory in any meaningful way — the inconsistent body list lives inside the PGS kernel solver and you'd need to walk that structure directly.

---

## Offsets

```cpp
namespace Offsets {
    // instance-relative
    uintptr_t DistributedGameTime           = 0x???;  // double
    uintptr_t FallenPartDestroyHeight       = 0x???;  // float
    uintptr_t ExperimentalSolverEnabled     = 0x???;  // bool
    uintptr_t ExpSolverEnabled_Replicate    = 0x???;  // bool
    uintptr_t NetworkStreamingEnabled       = 0x???;  // bool
    uintptr_t NetworkFilteringEnabled       = 0x???;  // bool
    uintptr_t AllowThirdPartySales          = 0x???;  // bool
    uintptr_t PhysicalPropertiesMode        = 0x???;  // int (enum)
    uintptr_t CurrentCamera                 = 0x???;  // shared_ptr<Camera> — read raw ptr inside
    uintptr_t RenderingDistance             = 0x???;  // float
    uintptr_t Terrain                       = 0x???;  // shared_ptr<Instance> — read raw ptr inside
    uintptr_t PhysicsAnalyzerBreakOnIssue   = 0x???;  // bool (inside PGS kernel)

    // module-relative (static members)
    uintptr_t showWorldCoordinateFrame      = 0x???;  // bool
    uintptr_t showHashGrid                  = 0x???;  // bool
    uintptr_t showEPhysicsOwners            = 0x???;  // bool
    uintptr_t showEPhysicsRegions           = 0x???;  // bool
    uintptr_t showStreamedRegions           = 0x???;  // bool
    uintptr_t showPartMovementPath          = 0x???;  // bool
    uintptr_t showActiveAnimationAsset      = 0x???;  // bool
    uintptr_t gridSizeModifier              = 0x???;  // float
}
```

all offsets are placeholders — dump them yourself, they shift between builds.

---

*more stuff coming as i dig further. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
