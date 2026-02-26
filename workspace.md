# Roblox Workspace System Docs

---

## What this covers

- distributed game time — how the simulation clock works and syncs
- physics settings — PGS solver, fallen parts, physical properties mode
- network flags — streaming, filtering, third party sales
- camera — how the current camera is managed and replenished
- terrain — how the MegaCluster terrain instance is stored and locked
- mouse command system — the tool/command stack and how input routes through it
- debug visualizers — the static flag set for rendering overlays
- physics analyzer — the PGS constraint inconsistency detector
- **how to read/write all of this yourself via offsets**

---

## Distributed Game Time

`distributedGameTime` is a `double` that tracks the current game simulation time in seconds. it's the clock everything runs on — physics steps, touch events, game logic. it is **not** the same as clock time or `os.time()`.

on the server it gets assigned from `RunService::gameTime()` each physics step. on the client it's updated but not broadcast back:

```cpp
void Workspace::updateDistributedGameTime()
{
    RunService* runService = ServiceProvider::create<RunService>(this);
    if (serverIsPresent(this)) {
        setDistributedGameTime(runService->gameTime());           // fires property change, replicates
    } else {
        setDistributedGameTimeNoTransmit(runService->gameTime()); // writes value silently, no event
    }
}
```

`setDistributedGameTime` calls `raiseChanged(prop_DistributedGameTime)` which fires a property change event and replicates the value to clients. `setDistributedGameTimeNoTransmit` just writes the double directly — no event, no replication. clients use the no-transmit path because they have no authority to broadcast time back.

`updateDistributedGameTime` is called at the start of every long physics step and also at `stop()` when the simulation ends.

the world extents cache also depends on this value — `computeExtentsWorldFast` only recomputes if more than 2 simulated seconds have passed since the last computation:

```cpp
if (distributedGameTime - lastComputedWorldExtentsTime > 2.0f)
{
    lastComputedWorldExtents = RootInstance::computeExtentsWorld();
    lastComputedWorldExtentsTime = distributedGameTime;
}
```

so if you freeze or manipulate this value, the extents cache will stop refreshing.

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

---

## Physics

### Fallen Parts

parts that fall below `FallenPartDestroyHeight` get collected each physics step by `world->computeFallen`, converted to `PartInstance*` via `primitivesToParts`, then handled based on network mode:

```cpp
void Workspace::handleFallenParts()
{
    world->computeFallen(fallenPrimitives);
    PartInstance::primitivesToParts(fallenPrimitives, fallenParts);

    if (Network::Players::getGameMode(this) == Network::DPHYS_CLIENT)
    {
        // client doesn't have authority — hand ownership to server instead of deleting
        for (size_t i = 0; i < fallenParts.size(); ++i)
            fallenParts[i]->setNetworkOwnerAndNotify(RBX::Network::NetworkOwner::Server());
    }
    else
    {
        // server has authority — delete them outright
        for (size_t i = 0; i < fallenParts.size(); ++i)
        {
            shared_ptr<Instance> oldParent = shared_from<Instance>(fallenParts[i]->getParent());
            fallenParts[i]->setParent(NULL);
            clearEmptiedModels(oldParent); // cleans up empty model containers
        }
    }
}
```

on a distributed physics client, fallen parts are never deleted locally — ownership transfers to the server which then handles deletion. on the server (or solo play), parts get destroyed immediately with `setParent(NULL)`.

after a part is removed, `clearEmptiedModels` walks up the parent chain and removes any `ModelInstance` or `Accoutrement` or `BackpackItem` containers that are now empty (and not a character). this is how the workspace automatically cleans up stale model containers.

`handleFallenParts` is called at the end of every physics step. `FallenPartDestroyHeight` is clamped to `[-50000, 50000]` by the setter. writing directly to the offset bypasses that clamp.

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

### PGS Solver (ExperimentalSolverEnabled)

there are actually two solver flags on the workspace that stay in sync:

```
experimentalSolverEnabled     — the workspace property, what you set in Studio
expSolverEnabled_Replicate    — the version that gets streamed to clients
```

the setters keep them locked together. setting `experimentalSolverEnabled` also updates `expSolverEnabled_Replicate`:

```cpp
void Workspace::setExperimentalSolverEnabled(bool value)
{
    if (experimentalSolverEnabled != value) {
        experimentalSolverEnabled = value;
        raiseChanged(prop_ExperimentalSolverEnabled);
    }
    if (expSolverEnabled_Replicate != experimentalSolverEnabled) {
        expSolverEnabled_Replicate = experimentalSolverEnabled;
        raiseChanged(prop_ExpSolverEnabled_Replicate);
    }
}
```

setting `expSolverEnabled_Replicate` syncs back to `experimentalSolverEnabled` AND tells the kernel:

```cpp
getWorld()->setUsingPGSSolver(expSolverEnabled_Replicate && FFlag::UsePGSSolver);
// master switch forces it on regardless of the workspace toggle
getWorld()->setUsingPGSSolver(getWorld()->getUsingPGSSolver() || FFlag::PGSAlwaysActiveMasterSwitch);
```

so the actual solver state lives in the kernel, not the workspace bools. there's a code FFlag `FFlag::UsePGSSolver` that gates the whole thing, and a `PGSAlwaysActiveMasterSwitch` that forces it on unconditionally, overriding both.

the Lua-accessible `ExperimentalSolverIsEnabled()` reads directly from `getWorld()->getKernel()->getUsingPGSSolver()` — not the workspace bool — so it reflects actual kernel state.

```cpp
bool getExperimentalSolverEnabled(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::ExperimentalSolverEnabled), &value, sizeof(bool), nullptr);
    return value;
}

void setExperimentalSolverEnabled(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ExperimentalSolverEnabled), &value, sizeof(bool), nullptr);
}

bool getExpSolverEnabled_Replicate(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::ExpSolverEnabled_Replicate), &value, sizeof(bool), nullptr);
    return value;
}

void setExpSolverEnabled_Replicate(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ExpSolverEnabled_Replicate), &value, sizeof(bool), nullptr);
}
```

> **note:** writing to the workspace bool offset alone won't activate the solver. the kernel has to be told separately via `setUsingPGSSolver`. if you want the solver actually running, write both the workspace bool and the kernel flag.

### PhysicalPropertiesMode

an enum controlling the physical material system for parts:

```
0 — Default
1 — Legacy
2 — New (NewPartProperties)
```

the setter has a runtime guard — it silently no-ops if the simulation is running or paused:

```cpp
void Workspace::setPhysicalPropertiesMode(PhysicalPropertiesMode mode)
{
    RunService* rs = ServiceProvider::find<RunService>(this);
    if (getWorld()->getPhysicalPropertiesMode() != mode)
    {
        if (rs && rs->getRunState() != RS_RUNNING && rs->getRunState() != RS_PAUSED)
        {
            getWorld()->setPhysicalPropertiesMode(mode);
            raiseChanged(prop_physicalPropertiesMode);
        }
        // silently ignored during runtime
    }
}
```

the value is stored on the `World` object, not the workspace. `getPhysicalPropertiesMode()` is just a passthrough to `getWorld()->getPhysicalPropertiesMode()`. there's a no-events variant (`setPhysicalPropertiesModeNoEvents`) used during file load that skips the runtime check entirely.

```cpp
// offset is on the World object, not workspace
int getPhysicalPropertiesMode(uintptr_t worldBase) {
    int value;
    ReadProcessMemory(hProcess, (LPVOID)(worldBase + Offsets::PhysicalPropertiesMode), &value, sizeof(int), nullptr);
    return value;
}

void setPhysicalPropertiesMode(uintptr_t worldBase, int value) {
    WriteProcessMemory(hProcess, (LPVOID)(worldBase + Offsets::PhysicalPropertiesMode), &value, sizeof(int), nullptr);
}
```

---

## Network Flags

### StreamingEnabled

enables gradual data streaming to clients instead of sending everything upfront. server-only, not replicated. the setter only raises a property change event in CloudEdit mode — outside of that, flipping this flag is silent and only takes effect on the next client connection:

```cpp
void Workspace::setNetworkStreamingEnabled(bool value)
{
    bool changed = value != networkStreamingEnabled;
    networkStreamingEnabled = value;
    if (changed && Network::Players::isCloudEdit(this))
        raisePropertyChanged(prop_StreamingEnabled);
}
```

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

the FE flag. when on, client-side changes to instances don't replicate to the server. the setter has two side effects beyond just writing the bool:

```cpp
void Workspace::setNetworkFilteringEnabled(bool value)
{
    // 1. fires a GA event the first time FE is enabled server-side (once per process)
    if (value && Workspace::serverIsPresent(this)) {
        static boost::once_flag flag = BOOST_ONCE_INIT;
        boost::call_once(&sendNetworkFilteringStats, flag);
    }

    networkFilteringEnabled = value;

    // 2. on the client, creates the local player's GUI if CreatePlayerGuiLocal DFFlag is active
    if (networkFilteringEnabled && DFFlag::CreatePlayerGuiLocal && Network::Players::frontendProcessing(this))
        if (Network::Player* player = Network::Players::findLocalPlayer(this))
            player->createPlayerGui();
}
```

the GA call fires exactly once per process lifetime via `boost::once_flag`. the PlayerGui creation only happens on the frontend (client side) when the DFFlag is set.

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

controls whether third-party game passes can be sold in the place. no complex side effects — just writes the bool and raises a property change in CloudEdit:

```cpp
void Workspace::setAllowThirdPartySales(bool value)
{
    bool changed = allowThirdPartySales != value;
    allowThirdPartySales = value;
    if (changed && Network::Players::isCloudEdit(this))
        raisePropertyChanged(prop_allowThirdPartySales);
}
```

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

the workspace manages two camera pointers:

```
currentCamera   — the active camera, lives in the workspace tree, used for rendering
utilityCamera   — an internal fallback, not in the tree, used when no camera child exists
```

`getCamera()` always returns `currentCamera` if set, otherwise falls back to `utilityCamera`. the result is never null.

### Camera Replenishment

`replenishCamera()` runs every heartbeat. it checks whether `currentCamera` is still a descendant of the workspace. if it's been deleted or reparented away, it searches for any `Camera` child and promotes it. if none exists, it clones `utilityCamera`, attaches it to the workspace, then sets it as current:

```cpp
void Workspace::replenishCamera()
{
    if (currentCamera && this->isAncestorOf(currentCamera.get()))
        return; // still valid, nothing to do

    shared_ptr<Camera> childCamera = shared_from<Camera>(findFirstChildOfType<Camera>());
    if (!childCamera) {
        childCamera = shared_polymorphic_downcast<Camera>(utilityCamera->clone(EngineCreator));
        childCamera->setParent(this);
    }
    setCurrentCamera(childCamera.get());
}
```

this means you can't permanently destroy the workspace camera — the engine recreates it next heartbeat.

### Setting the Current Camera

setting `currentCamera` is only respected on the client or in CloudEdit — the server ignores it:

```cpp
void Workspace::setCurrentCamera(Camera* value)
{
    if (!serverIsPresent(this) || Network::Players::isCloudEdit(this)) {
        if (value != currentCamera.get()) {
            currentCamera = shared_from<Camera>(value);
            this->raisePropertyChanged(currentCameraProxyProp);
            if (value)
                visitChildren(boost::bind(&destroyIfNotCurrent, _1, value)); // destroys any other cameras
            currentCameraChangedSignal(currentCamera);
        }
    }
}
```

when you set a new camera it also visits all workspace children and destroys any other `Camera` instances — so setting current camera is destructive to siblings.

### RenderingDistance

a plain `float`, initialized to `10000.0f` in the constructor. no clamping or side effects — just read by the renderer.

```cpp
uintptr_t getCurrentCamera(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::CurrentCamera), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // raw Camera*
}

void setCurrentCamera(uintptr_t base, uintptr_t cameraPtr) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::CurrentCamera), &cameraPtr, sizeof(uintptr_t), nullptr);
}

float getRenderingDistance(uintptr_t base) {
    float value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::RenderingDistance), &value, sizeof(float), nullptr);
    return value;
}

void setRenderingDistance(uintptr_t base, float value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::RenderingDistance), &value, sizeof(float), nullptr);
}
```

> **note:** `currentCamera` is a `shared_ptr<Camera>` internally. on MSVC the layout is `[px][pn]` (each 8 bytes) — the raw pointer is at the offset directly. if reads come back garbage, try `offset + 0x8`.

---

## Terrain

terrain is a `shared_ptr<Instance>` pointing to a `MegaClusterInstance`. when set, its parent is immediately locked so it can't be reparented or destroyed through normal means:

```cpp
void Workspace::setTerrain(Instance* terrain)
{
    this->terrain = shared_from(terrain);
    this->raisePropertyChanged(workspace_Terrain);
    if (terrain) {
        this->terrain->lockParent();
        this->terrain->setLockedParent(this);
    }
}
```

`createTerrain()` only creates a terrain instance if one doesn't already exist. it positions the MegaCluster at `(-2, rbxSize.y/2, -2)` — the slight offset is intentional for voxel grid alignment. it uses `setAndLockParent` which sets the parent and locks it in one call.

`clearTerrain()` calls `unlockParent()` first before nulling the pointer — necessary because the lock would otherwise prevent removal.

the heartbeat also enforces terrain's parent every tick — if `terrain->getParent() != this` for any reason, it forcibly re-attaches it:

```cpp
if (terrain && terrain->getParent() != this)
    terrain->setAndLockParent(this);
```

so any external attempt to change terrain's parent will be corrected within one heartbeat.

```cpp
uintptr_t getTerrain(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Terrain), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // raw Instance* (MegaClusterInstance)
}
```

> **note:** terrain is effectively read-only via memory. writing a pointer here skips the parent lock setup, fires no property change, and gets corrected next heartbeat anyway. use the pointer to walk the MegaCluster structure, not to replace it.

---

## Mouse Command System

the workspace owns the active mouse command (tool). two slots:

```
currentCommand   — the command currently receiving input
stickyCommand    — the last "sticky" command — restored after a non-sticky command finishes
```

when a command returns null from its handler (signaling it's done), `setMouseCommand(nullptr)` is called. the workspace resolves the next command in this order:

```cpp
// 1. check if stickyCommand has a valid copy to restore
if (stickyCommand.get())
    newMouseCommand = stickyCommand.get()->isSticky(); // returns a clone if still sticky

// 2. no sticky — pick a default based on context
if (newMouseCommand.get() == NULL) {
    if (findLocalPlayer == NULL || isCloudEdit)
        newMouseCommand = create<AdvArrowTool>(this); // Studio: arrow tool
    else
        newMouseCommand = newNullTool(this);          // gameplay: null tool
}
```

a command is "sticky" if `isSticky()` returns a non-null copy of itself. this is how the arrow tool stays active between operations.

plugin override: if a plugin is active as a tool, `setMouseCommand` rejects any new command unless `allowPluginOverride = true`. activating a new camera through `setCurrentCamera` will also deactivate the plugin tool via `PluginManager::activate(NULL, dataModel)`.

input routing: every event goes through `handleSurfaceGui` first. if a `SurfaceGui` sinks the event, the current mouse command never sees it. only unsunk events reach `currentCommand`.

---

## Debug Visualizers

all of the `show*` fields below are **static bools** on the `Workspace` class — they're process-wide, not per-instance. read/write them at module offsets, not relative to a workspace instance.

```
showWorldCoordinateFrame  — renders XYZ axes at world origin
showHashGrid              — renders an AABox at the spatial hash region (hardcoded at pos (28,4,12) size (4,4,4))
showEPhysicsOwners        — highlights physics ownership per part
showEPhysicsRegions       — highlights physics region boundaries
showStreamedRegions       — highlights streamed client regions
showPartMovementPath      — shows recorded movement paths for parts
showActiveAnimationAsset  — overlays current animation asset info
gridSizeModifier          — float, scales 3D grid density (default 4.0f)
```

`show3DGrid` and `showAxisWidget` are different — those are **instance** fields, per-workspace. `show3DGrid` also has a context check: it won't render in a live game session even if true, only in Studio/CloudEdit:

```cpp
if (show3DGrid && (!localPlayer || Network::Players::isCloudEdit(this)))
    RBX::DrawAdorn::zeroPlaneGrid(adorn, *getCamera(), gridSizeModifier, 0.05, ...);
```

```cpp
// static fields — module base, not workspace instance base
bool getShowEPhysicsOwners(uintptr_t moduleBase) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(moduleBase + Offsets::showEPhysicsOwners), &value, sizeof(bool), nullptr);
    return value;
}

void setShowEPhysicsOwners(uintptr_t moduleBase, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(moduleBase + Offsets::showEPhysicsOwners), &value, sizeof(bool), nullptr);
}

// instance fields — workspace base
bool getShow3DGrid(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::show3DGrid), &value, sizeof(bool), nullptr);
    return value;
}

void setShow3DGrid(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::show3DGrid), &value, sizeof(bool), nullptr);
}
```

---

## Physics Analyzer

the physics analyzer detects constraint solver failures in the PGS kernel — bodies where the solver couldn't find a consistent solution between colliding parts.

it only runs when `FFlag::PhysicsAnalyzerEnabled` is set and PGS is active. each step, after `world->step()` finishes:

```cpp
if (FFlag::PhysicsAnalyzerEnabled && world->getKernel()->pgsSolver.getInconsistentBodyPairs().size() > 0)
    luaPhysicsAnalyzerIssuesFound(world->getKernel()->pgsSolver.getInconsistentBodies().size());
```

this fires the Lua `PhysicsAnalyzerIssuesFound` event with the inconsistent body count. from Lua, `GetPhysicsAnalyzerIssue(group)` walks the body list and returns the `PartInstance` objects in a given group — but only if `BreakOnIssue` is enabled.

`SetPhysicsAnalyzerBreakOnIssue` is a full no-op if PGS isn't enabled — the setter checks `getExperimentalSolverEnabled()` first and returns silently if it's false:

```cpp
void Workspace::setPhysicsAnalyzerBreakOnIssue(bool enable)
{
    if (getExperimentalSolverEnabled())
        getWorld()->getKernel()->pgsSolver.setPhysicsAnalyzerBreakOnIssue(enable);
}
```

the actual flag lives inside `pgsSolver` in the kernel, not on the workspace.

```cpp
bool getPhysicsAnalyzerBreakOnIssue(uintptr_t kernelBase) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(kernelBase + Offsets::PhysicsAnalyzerBreakOnIssue), &value, sizeof(bool), nullptr);
    return value;
}

void setPhysicsAnalyzerBreakOnIssue(uintptr_t kernelBase, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(kernelBase + Offsets::PhysicsAnalyzerBreakOnIssue), &value, sizeof(bool), nullptr);
}
```

---

## Offsets

```cpp
namespace Offsets {
    // workspace instance base
    uintptr_t DistributedGameTime           = 0x???;  // double
    uintptr_t FallenPartDestroyHeight       = 0x???;  // float
    uintptr_t ExperimentalSolverEnabled     = 0x???;  // bool
    uintptr_t ExpSolverEnabled_Replicate    = 0x???;  // bool
    uintptr_t NetworkStreamingEnabled       = 0x???;  // bool
    uintptr_t NetworkFilteringEnabled       = 0x???;  // bool
    uintptr_t AllowThirdPartySales          = 0x???;  // bool
    uintptr_t CurrentCamera                 = 0x???;  // shared_ptr<Camera> — raw ptr at offset
    uintptr_t RenderingDistance             = 0x???;  // float (default 10000.0f)
    uintptr_t Terrain                       = 0x???;  // shared_ptr<Instance> — raw ptr at offset
    uintptr_t show3DGrid                    = 0x???;  // bool (instance field)
    uintptr_t showAxisWidget                = 0x???;  // bool (instance field)

    // world base (workspace->world)
    uintptr_t PhysicalPropertiesMode        = 0x???;  // int (0=Default, 1=Legacy, 2=New)

    // kernel base (workspace->world->kernel->pgsSolver)
    uintptr_t PhysicsAnalyzerBreakOnIssue   = 0x???;  // bool

    // module base (static fields — not instance offsets)
    uintptr_t showWorldCoordinateFrame      = 0x???;  // bool
    uintptr_t showHashGrid                  = 0x???;  // bool
    uintptr_t showEPhysicsOwners            = 0x???;  // bool
    uintptr_t showEPhysicsRegions           = 0x???;  // bool
    uintptr_t showStreamedRegions           = 0x???;  // bool
    uintptr_t showPartMovementPath          = 0x???;  // bool
    uintptr_t showActiveAnimationAsset      = 0x???;  // bool
    uintptr_t gridSizeModifier              = 0x???;  // float (default 4.0f)
}
```

all offsets are placeholders — dump them yourself. they shift between builds.

---

*more stuff coming as i dig further. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
