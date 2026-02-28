# Roblox TeleportService System Docss

---

## What this covers

- the three teleport types and how they map to PlaceLauncher URLs
- the server vs client execution split
- `TeleportImpl` — what happens when a client initiates a teleport
- the retry/polling thread (`TeleportThreadImpl`) and its status codes
- `ServerTeleport` — how server-side teleport works
- `TeleportToPrivateServer` and the `ReserveServer` grant access flow
- `GetPlayerPlaceInstanceAsync`
- teleport state lifecycle
- custom loading GUIs — how they're sanitized and replicated
- settings persistence across teleports (`SetTeleportSetting` / `GetTeleportSetting`)
- static state — what survives after a teleport

---

## Teleport Types

there are three types, defined in the `TeleportType` enum:

| Value | Name | PlaceLauncher request param |
|---|---|---|
| `0` | `ToPlace` | `RequestGame` |
| `1` | `ToInstance` | `RequestGameJob` |
| `2` | `ToReservedServer` | `RequestPrivateGame` |

each type produces a different PlaceLauncher URL in `TeleportImpl`:

```cpp
// ToPlace
"Game/PlaceLauncher.ashx?request=RequestGame&placeId=%d&isPartyLeader=false&gender=&isTeleport=true"

// ToInstance
"Game/PlaceLauncher.ashx?request=RequestGameJob&placeId=%d&gameId=%s&isPartyLeader=false&gender=&isTeleport=true"

// ToReservedServer
"Game/PlaceLauncher.ashx?request=RequestPrivateGame&placeId=%d&accessCode=%s&linkCode=&privateGameMode=ReservedServer"
```

if `_browserTrackerId` is set, it gets appended to all of these as `&browserTrackerId=...`.

---

## Teleport State Lifecycle

```
TeleportState_RequestedFromServer  (0)  — server told client to teleport
TeleportState_Started              (1)  — client built the URL and spawned the thread
TeleportState_WaitingForServer     (2)  — PlaceLauncher returned status 0 or 1
TeleportState_InProgress           (3)  — PlaceLauncher returned status 2, auth ticket received
TeleportState_Failed               (4)  — retries exhausted or exception thrown
```

state transitions fire `player->onTeleportInternal(state, teleportInfo)` so the player object tracks them. the signal `LocalPlayerArrivedFromTeleport` fires on arrival in the new place with the loading GUI and data table.

---

## Server vs Client Split

**every public teleport method checks `Workspace::serverIsPresent(this)`** and routes accordingly:

```cpp
bool isServer = Workspace::serverIsPresent(this);
if (isServer) {
    ServerTeleport(characterOrPlayerInstance, teleportInfo, customLoadingGUI);
    return;
}
// client path
TeleportImpl(teleportInfo, customLoadingGUI);
```

`Teleport` and `TeleportToSpawnByName` and `TeleportToPlaceInstance` all follow this exact pattern. `TeleportToPrivateServer` is server-only by assertion.

---

## Client Path: TeleportImpl

this is what actually kicks off a client-side teleport.

```cpp
void TeleportService::TeleportImpl(
    shared_ptr<const Reflection::ValueTable> teleportInfo,
    shared_ptr<Instance> customLoadingGUI)
```

**what it does, in order:**

1. asserts we're not on the server (`RBXASSERT(!Workspace::serverIsPresent(this))`)
2. saves `teleportData` and `customTeleportLoadingGui` to static storage
3. strips all `LuaSourceContainer` descendants from the custom loading GUI and unparents it
4. returns early if `requestingTeleport` is already true (no double-teleports)
5. checks `_callback->isTeleportEnabled()` — if disabled (e.g. in Studio), fires the error callback and returns
6. builds the PlaceLauncher URL based on `teleportType`
7. fires `TeleportState_Started` on the local player
8. resets the retry timer and spawns `TeleportThreadImpl` on a new thread

**the `requestingTeleport` flag is what prevents concurrent teleports.** it's set to `true` here and only cleared on failure or cancellation.

---

## The Retry Thread: TeleportThreadImpl

this runs on a background thread and polls PlaceLauncher until it gets a server to connect to.

```cpp
int retries = DFInt::TeleportRetryTimes; // default 5
bool cancelable = true;

while (true) {
    // POST (or GET) to the PlaceLauncher URL
    // parse JSON response, check "status" field
}
```

### status codes

| Status | Meaning | Action |
|---|---|---|
| `2` | server found | extract auth ticket + join script URL, call `_callback->doTeleport(...)` |
| `0` or `1` | server not ready yet | fire `WaitingForServer`, mark `cancelable = false`, don't decrement retries |
| anything else | error | decrement `retries` |

once `cancelable` is set to `false` (status 0/1 or 2), `TeleportCancel()` can no longer stop the teleport.

**retry timing:** after each loop iteration, the thread sleeps for however long is left of a 1-second window (`1000ms - elapsed`). so retries happen at most once per second.

**retry exhaustion:** when `retries < 0`, fires `TeleportState_Failed` and returns.

### on success (status 2)

```cpp
std::string au     = ReadStringValue(jsonResult, "authenticationUrl");
std::string ticket = ReadStringValue(jsonResult, "authenticationTicket");
std::string script = ReadStringValue(jsonResult, "joinScriptUrl");  // or constructed in Studio

previousPlaceId     = dm->getPlaceID();
previousCreatorId   = dm->getCreatorID();
previousCreatorType = dm->getCreatorType();
teleported = true;

_callback->doTeleport(au, ticket, script);
```

the previous place/creator info is stashed in static `HeapValue<int>` fields so the new place can read them after loading.

**Studio mode** is special-cased: instead of reading `joinScriptUrl` from the response, it hits `/universes/get-universe-containing-place` to get the universe ID and constructs a `Visit.ashx` URL itself.

---

## Server Path: ServerTeleport

```cpp
void TeleportService::ServerTeleport(
    shared_ptr<Instance> characterOrPlayerInstance,
    shared_ptr<Reflection::ValueTable> teleportInfo,
    shared_ptr<Instance> customLoadingGUI)
```

the instance can be either a `Player` or a character `Model`. passing a character is supported but deprecated — it logs a warning and resolves the player via `Players::getPlayerFromCharacter`.

once a valid player is found:
1. fires GA stats for the teleport type (once per process via `boost::call_once`)
2. sanitizes the custom loading GUI (see below)
3. parents the loading GUI to `InsertService` temporarily so it can replicate to the client
4. fires `TeleportState_RequestedFromServer` on the player via `onTeleportInternal`
5. unparents the loading GUI

the actual network handoff happens inside `onTeleportInternal` on the player object, not here.

---

## Custom Loading GUI Sanitization

```cpp
static shared_ptr<Instance> sanitizeCustomLoadingGui(shared_ptr<Instance> customLoadingGUI)
{
    if (customLoadingGUI) {
        shared_ptr<Instance> sanitizedCustomLoadingGUI = customLoadingGUI->clone(EngineCreator);
        customLoadingGUI->destroyDescendantsOfType<LuaSourceContainer>();
        return sanitizedCustomLoadingGUI;
    }
    return shared_ptr<Instance>();
}
```

it clones the GUI first, then strips all `LuaSourceContainer` descendants from the **original**. the sanitized (script-free) copy is what gets used/replicated. this prevents scripts from running in the loading screen during the transition.

on the client side (`TeleportImpl`), the GUI also gets unparented (`setParent(NULL)`) before the teleport proceeds.

---

## TeleportToPrivateServer

server-only. requires a `reservedServerAccessCode` from `ReserveServer`.

```cpp
void TeleportService::TeleportToPrivateServer(
    int placeId,
    std::string reservedServerAccessCode,
    shared_ptr<const Instances> players,
    std::string spawnName,
    Reflection::Variant teleportData,
    shared_ptr<Instance> customLoadingGUI)
```

builds a player ID list (`playerIds=X&playerIds=Y&...`) and POSTs to:

```
reservedservers/grantaccess?reservedServerAccessCode=<encoded_code>
```

on success (`ProcessGrantAccessSuccess`), calls `ServerTeleport` for each player individually. on failure (`ProcessGrantAccessError`), fires `TeleportState_Failed` on each player.

---

## ReserveServer

```cpp
void TeleportService::ReserveServer(int placeId, resumeFunction, errorFunction)
```

server-only. POSTs to:

```
reservedservers/create?placeId=<placeId>
```

parses the JSON response for `ReservedServerAccessCode` (a string). resumes the Lua coroutine with the access code on success, or calls the error function with a message on failure. all callbacks are submitted back to the DataModel write task queue.

---

## GetPlayerPlaceInstanceAsync

```cpp
void TeleportService::GetPlayerPlaceInstanceAsync(int playerId, resumeFunction, errorFunction)
```

GETs:

```
universes/get-player-place-instance?currentPlaceId=<placeId>&userId=<playerId>
```

returns a 4-tuple on success: `(success: bool, errorMessage: string, placeId: int, gameId: string)`. if parsing fails for any reason, calls the error function instead.

---

## Teleport Settings

a static `unordered_map<string, Variant>` that persists for the lifetime of the process:

```cpp
void TeleportService::SetTeleportSetting(std::string key, Reflection::Variant value) {
    settingsTable[key] = value;
}

Reflection::Variant TeleportService::GetTeleportSetting(std::string key) {
    auto iter = settingsTable.find(key);
    if (iter != settingsTable.end())
        return iter->second;
    return Reflection::Variant(); // nil
}
```

because it's static, these settings survive teleports. that's the point — you set something before teleporting and read it back in the new place. if a key doesn't exist, you get back an empty `Variant` (nil in Lua).

---

## Static State That Survives Teleport

these static fields are written just before `_callback->doTeleport(...)` fires:

```cpp
static HeapValue<int> previousPlaceId;      // dm->getPlaceID()
static HeapValue<int> previousCreatorId;    // dm->getCreatorID()
static HeapValue<int> previousCreatorType;  // dm->getCreatorType()
static bool teleported;                     // set to true

static boost::shared_ptr<Instance> customTeleportLoadingGui;
static Reflection::Variant dataTable;       // the teleportData passed in
static SettingsMap settingsTable;           // SetTeleportSetting values
```

the new place reads these through the static getters (`getPreviousPlaceId()`, `didTeleport()`, `getDataTable()`, etc.) during its loading sequence.

---

## TeleportCancel

```cpp
void TeleportService::TeleportCancel() {
    _waitingForUserInput = false;
    requestingTeleport = false;
}
```

only works if the thread hasn't set `cancelable = false` yet — which happens as soon as PlaceLauncher returns status 0, 1, or 2. after that point, the teleport will complete regardless. this is intentional: once the web service has been contacted and might be disconnecting the player, there's no safe cancel.

---

## teleportInfo Table Keys

every internal teleport packages its parameters into a `ValueTable` before passing it around:

| Key | Type | Notes |
|---|---|---|
| `placeId` | `int` | destination place |
| `spawnName` | `string` | spawn location name, empty string for default |
| `instanceId` | `string` | game job ID, only used for `ToInstance` |
| `reservedServerAccessCode` | `string` | only used for `ToReservedServer` |
| `teleportType` | `TeleportType` | enum value |
| `teleportData` | `Variant` | arbitrary Lua data passed by the caller |

---

*I plan to look into this service more*
