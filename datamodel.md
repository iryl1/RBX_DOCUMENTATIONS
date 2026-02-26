# Roblox DataModel System Docs

 this one covers the DataModel — what it actually is, how it gets created and destroyed, how its services are initialized, the hack flag system, threading model, and all the metadata it carries (placeId, creatorId, genre, etc.).


---
- note this one was one of the hardest 
## What this covers

- what DataModel actually is and what it inherits from
- how DataModel instances work (there are multiple)
- the full service initialization order
- place/game metadata (placeId, universeId, creatorId, genre, gear settings)
- the hack flag system (sendStats, perfStats)
- threading model (LegacyLock, scoped_write_request, GenericJob)
- lifecycle (create → init → load → close)
- shutdown sequence in detail
- enums (CreatorType, Genre, GearType, GearGenreSetting)
- default values from the constructor
- what to read from memory for cheat dev purposes

---

## What DataModel actually is

DataModel is the root container for everything in a Roblox game session. it inherits from `ServiceProvider`, which means it owns and manages all the game services (Workspace, Players, Lighting, etc.) as children. its RTTI string is `"DataModel"`.

one thing that trips people up — **multiple DataModel instances exist at the same time**. they each serve a different internal purpose (rendering context, networking context, studio, etc.). the one that actually owns all the game services and has `Workspace` and `Players` hanging off of it is what the community calls the "FakeDataModel" — that name is just a historical accident and is misleading. it's the real one.

to find it: scan for all pointers with RTTI `DataModel@RBX`, sort by address, grab index `[1]` (second-largest). that address contains a pointer to the described DataModel. you read the address, then add an offset and read again to get the actual instance pointer. from there you can traverse the service tree like you would in Lua.

the name `"DataModel"` is the internal class name but Lua exposes it as `game` — the constructor literally calls `Super("Game")` so its instance name is `"Game"`.

---

## Lifecycle

### creation

```
DataModel::createDataModel(startHeartbeat, lockVerb, shouldShowLoadingScreen)
  → allocates DataModel
  → doDataModelSetup()
    → creates GenericJobs (Write Marshalled, Read Marshalled, None Marshalled)
    → acquires LegacyLock (Write)
    → initializeContents(startHeartbeat)   ← all services created here
    → isInitialized = true
    → if shouldShowLoadingScreen → executes LoadingScript corescript
```

### initialization order (initializeContents)

these services are created in this exact order — worth knowing if you're traversing the service tree:

```
Workspace                      ← created in constructor, parented here
GuiRoot                        ← same
NonReplicatedCSGDictionaryService
CSGDictionaryService
LogService
ContentProvider
ContentFilter
KeyframeSequenceProvider
GuiService
ChatService
MarketplaceService
PointsService
AdService
NotificationService
ReplicatedFirst
HttpRbxApiService
StarterPlayerService
StarterPackService
StarterGuiService
CoreGuiService                 ← also calls createRobloxScreenGui()
TeleportService                ← checks for custom teleport loading GUI
RunService                     ← starts heartbeat if startHeartbeat is true
SoundService
JointsService
CollectionService
PhysicsService
BadgeService
GeometryService
FriendService
RenderHooksService
InsertService
SocialService
GamePassService
DebrisService
ScriptInformationProvider
CookiesService
TeleportService                ← yes, created twice, not a typo
PersonalServerService
Players                        ← always created even without a server
UserInputService
ContextActionService
ScriptService
AssetService
```

### close sequence

```
doCloseDataModel()
  → disables ChangeHistoryService
  → fires closingSignal / closingLateSignal
  → isInitialized = false
  → Players::removeAllChildren()
  → clearContents(false)
  → visitChildren(unlockParent)
  → removeAllChildren()
  → workspace.reset()
  → runService.reset()
  → starterPackService.reset()
  → starterGuiService.reset()
  → starterPlayerService.reset()
  → guiRoot.reset()
  → clearServices()
  → removes all GenericJobs from TaskScheduler
```

if `onCloseCallback` is set (the Lua `game.OnClose` callback), the close sequence waits up to `DFInt::OnCloseTimeoutInSeconds` (default 30 seconds) for it to finish before proceeding.

---

## Threading model

DataModel has a fairly involved threading system. the short version:

**GenericJob** — there are three background jobs always running for a DataModel:
- `Write Marshalled` — for write tasks
- `Read Marshalled` — for read tasks
- `None Marshalled` — for misc tasks

**LegacyLock** — used to marshal work from any thread onto one of the GenericJobs in a thread-safe way. when you acquire a LegacyLock, your thread blocks until the scheduler picks it up. re-entrant (if you're already on the same job, it's a no-op).

```cpp
// acquiring write access from an arbitrary thread:
DataModel::LegacyLock lock(dataModel, DataModelJob::Write);
// now safe to read/write the DataModel
```

**scoped_write_request / scoped_read_request** — RAII guards that set/clear `write_requested` and `read_requested` flags. these exist for debugging concurrency violations, not actual locking.

`currentThreadHasWriteLock()` checks if the current thread is the one that acquired the write lock. useful for asserts.

for cheat dev purposes: the DataModel holds `volatile long write_requested` and `volatile DWORD writeRequestingThread` — these tell you what thread currently owns the write lock and whether it's active.

---

## Hack flag system

this is roblox's built-in anti-cheat detection tracking. two static bitmasks:

```cpp
static unsigned int DataModel::sendStats;  // legacy bitmask — flagged if hacking detected
static unsigned int DataModel::perfStats;  // secondary bitmask
```

flags are added/removed via `addHackFlag(flag)` and `removeHackFlag(flag)`, protected by a mutex. the flags are also aggregated in an `unordered_set<unsigned int>` called `hackFlagSet`. `isHackFlagSet(flag)` checks both the set and the bitmask.

`sendStats` is a static — it's shared across all DataModel instances. when a hack flag is set, it stays in `sendStats` even if it's removed from `hackFlagSet`. the names of the specific flags are in `HackDefines.h` (not included here).

from a memory perspective: `sendStats` is a static so you can find it by its address in the module rather than as an offset from a DataModel instance.

---

## Place and game metadata

all of these are stored directly on the DataModel instance:

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `placeID` | `HeapValue<int>` | `0` | the place's asset ID |
| `placeVersion` | `int` | `0` | version number |
| `universeId` | `int` | `0` | parent universe ID |
| `gameInstanceID` | `std::string` | `""` | job GUID, also exposed as `JobId` |
| `creatorID` | `HeapValue<int>` | `0` | user or group ID |
| `creatorType` | `CreatorType` | `CREATOR_USER` | user or group |
| `genre` | `Genre` | not set in ctor | game genre |
| `gearGenreSetting` | `GearGenreSetting` | not set in ctor | gear genre restriction |
| `allowedGearTypes` | `unsigned int` | not set in ctor | bitmask, see GearType enum |
| `vipServerId` | `std::string` | `""` | VIP server ID if applicable |
| `vipServerOwnerId` | `int` | `0` | owner of the VIP server |
| `isPersonalServer` | `bool` | `false` | |
| `forceR15` | `bool` | `false` | forces R15 rig for all characters |
| `runningInStudio` | `bool` | `false` | |
| `isStudioRunMode` | `bool` | `false` | true when in studio run mode |
| `remoteBuildMode` | `bool` | `false` | always false, setter is a no-op |

### other constructor defaults

```cpp
shutdownRequestedCount = 0
write_requested        = 0
read_requested         = 0
dirty                  = false
isInitialized          = false
isContentLoaded        = false
isShuttingDown         = false
drawId                 = 0
physicsStepID          = 0
totalWorldSteps        = 0
numPartInstances       = 0
numInstances           = 0
mouseOverGui           = false
networkMetric          = nullptr
isGameLoaded           = false
areCoreScriptsLoaded   = false
networkStatsWindowsOn  = false
forceArrowCursor       = true
canRequestUniverseInfo = false
universeDataRequested  = false
renderGuisActive       = DFFlag::AllowHideHudShortcutDefault  // usually true
```

---

## Enums

### CreatorType

| Value | Name |
|-------|------|
| `0` | `User` |
| `1` | `Group` |

### Genre

| Value | Name |
|-------|------|
| `0` | `All` |
| `1` | `TownAndCity` |
| `2` | `Fantasy` |
| `3` | `SciFi` |
| `4` | `Ninja` |
| `5` | `Scary` |
| `6` | `Pirate` |
| `7` | `Adventure` |
| `8` | `Sports` |
| `9` | `Funny` |
| `10` | `WildWest` |
| `11` | `War` |
| `12` | `SkatePark` |
| `13` | `Tutorial` |

### GearGenreSetting

| Value | Name |
|-------|------|
| `0` | `AllGenres` |
| `1` | `MatchingGenreOnly` |

### GearType (bitmask — `allowedGearTypes & (1 << gearType)`)

| Bit | Name |
|-----|------|
| `0` | `MeleeWeapons` |
| `1` | `RangedWeapons` |
| `2` | `Explosives` |
| `3` | `PowerUps` |
| `4` | `NavigationEnhancers` |
| `5` | `MusicalInstruments` |
| `6` | `SocialItems` |
| `7` | `BuildingTools` |
| `8` | `Transport` |

### RequestShutdownResult

| Value | Name |
|-------|------|
| `0` | `CLOSE_NOT_HANDLED` |
| `1` | `CLOSE_REQUEST_HANDLED` |
| `2` | `CLOSE_LOCAL_SAVE` |
| `3` | `CLOSE_NO_SAVE_NEEDED` |

---

## Signals worth knowing about

| Signal | When it fires |
|--------|---------------|
| `gameLoadedSignal` / `Loaded` | when `setIsGameLoaded(true)` is called |
| `workspaceLoadedSignal` | after `loadContent` finishes and terrain is created |
| `itemChangedSignal` | any time any property on any instance changes |
| `allowedGearTypeChanged` | when allowed gear types are updated |
| `screenshotSignal` | when a screenshot is taken |
| `saveFinishedSignal` | after `save()` completes |
| `closingSignal` / `closingLateSignal` | fired inside `raiseClose()` during shutdown |

---

## Reading from memory

DataModel doesn't have the kind of offset-based replication that Humanoid or Lighting do — it's more of a metadata container you read from rather than write to. the most useful things to read:

```cpp
namespace Offsets {
    // ints
    uintptr_t PlaceID        = 0x???;  // HeapValue<int>
    uintptr_t PlaceVersion   = 0x???;  // int
    uintptr_t UniverseId     = 0x???;  // int
    uintptr_t CreatorID      = 0x???;  // HeapValue<int>
    uintptr_t CreatorType    = 0x???;  // int (0 = user, 1 = group)
    uintptr_t Genre          = 0x???;  // int (see enum above)
    uintptr_t VipServerOwner = 0x???;  // int

    // bools
    uintptr_t IsInitialized    = 0x???;
    uintptr_t IsContentLoaded  = 0x???;
    uintptr_t IsShuttingDown   = 0x???;
    uintptr_t IsGameLoaded     = 0x???;
    uintptr_t RunningInStudio  = 0x???;
    uintptr_t ForceR15         = 0x???;
    uintptr_t IsPersonalServer = 0x???;

    // counters
    uintptr_t NumPartInstances = 0x???;  // int
    uintptr_t NumInstances     = 0x???;  // int
    uintptr_t PhysicsStepID    = 0x???;  // int, increments every physics tick
    uintptr_t TotalWorldSteps  = 0x???;  // int
}

HANDLE hProcess = nullptr;
```

reading examples:

```cpp
int ReadPlaceID(uintptr_t dmBase) {
    int val = 0;
    ReadProcessMemory(hProcess, (LPCVOID)(dmBase + Offsets::PlaceID), &val, sizeof(int), nullptr);
    return val;
}

bool ReadIsShuttingDown(uintptr_t dmBase) {
    bool val = false;
    ReadProcessMemory(hProcess, (LPCVOID)(dmBase + Offsets::IsShuttingDown), &val, sizeof(bool), nullptr);
    return val;
}

int ReadPhysicsStepID(uintptr_t dmBase) {
    int val = 0;
    ReadProcessMemory(hProcess, (LPCVOID)(dmBase + Offsets::PhysicsStepID), &val, sizeof(int), nullptr);
    return val;
}
```

`PhysicsStepID` is useful as a reliable tick counter — it increments every physics step regardless of frame rate. `NumPartInstances` and `NumInstances` are handy for quick sanity checks that you're reading the right DataModel.

for `sendStats` — since it's a static, you find it by scanning for the pattern in the module rather than reading from a DataModel instance offset. when it's non-zero, the game has already flagged the client as suspicious.

---

*more systems coming. lmk if anything's wrong or missing.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
