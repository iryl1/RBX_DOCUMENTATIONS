# Roblox Players System Docs

`Players` is the service that manages the player list, handles chat routing, tracks local player state, and runs the anti tamper hash checks


---

## What this covers

- fields — what's stored on the service
- the player list — how it's maintained and how to iterate it
- local player — creation, reset, and the dangerous getter
- game mode detection — `clientIsPresent`, `serverIsPresent`, `frontendProcessing`, `backendProcessing`, and `getGameMode`
- `onChildAdded` / `onChildRemoving` — what happens when a player joins or leaves
- chat — routing, the history, super safe chat, the `/sc` command parser
- CharacterAutoLoads
- block/unblock — the client/server flow
- the anti-cheat hash system — golden hashes, mem hashes, and when kicks happen
- **how to read/write all of this yourself via offsets**

---

## Fields

```
players                  — copy_on_write_ptr<Instances> (the actual player list)
rakPeer                  — ConcurrentRakPeer* (network connection)
maxPlayers               — int (default 12)
preferredPlayers         — int
localPlayer              — shared_ptr<Player> (null on server)
chatOption               — ChatOption enum (Classic=0, Bubble=1, ClassicAndBubble=2)
characterAutoSpawn       — bool (default true)
testPlayerNameId         — int (counter for test player names)
testPlayerUserId         — int (counter, negative, for test player userids)
chatHistory              — std::list<ChatMessage>
guidRegistry             — boost::intrusive_ptr<GuidItem<Instance>::Registry>
abuseReporter            — scoped_ptr<AbuseReporter> (null until setAbuseReportUrl called)
nonSuperSafeChatForAllPlayersEnabled — bool
saveDataUrl / loadDataUrl / saveLeaderboardDataUrl — std::string
chatFilterUrl / buildUserPermissionsUrl / sysStatsUrl — std::string
goldenHash / goldenHash2 / goldenHash3 — std::string (instance, per-Players object)
leaderboardKeys          — boost::unordered_set<std::string>
cheatingPlayers          — std::map<int, std::set<std::string>> (exploit detection log)
```

static fields:
```
canKickBecauseRunningInRealGameServer — bool (set when any golden hash is registered)
goldenHashes              — std::set<std::string> (the full hash allowlist, mutex-guarded)
goldMemHashes             — MemHashConfigs (memory hash configs for anti-cheat)
```

---

## Player List

`players` is a `copy_on_write_ptr<Instances>` — the same pattern as Instance children but maintained separately. it's always initialized to a non-null empty vector (done in the constructor: `players(Instances())`), so reading it won't give you a null pointer like Instance children can.

the list is updated in `onChildAdded` and `onChildRemoving` — when a Player is added as a child of Players service, it's pushed into this vector. when it's removed, it's erased from the vector. the child list and the players vector are kept in sync.

`getPlayers()` returns `players.read()` — a `shared_ptr<const Instances>`.

to look up by user ID, the engine iterates the vector linearly:

```cpp
shared_ptr<Player> Players::getPlayerByID(int userID)
{
    for (auto iter = players->begin(); iter != players->end(); ++iter)
    {
        shared_ptr<Player> player = shared_polymorphic_downcast<Player>(*iter);
        if (player->getUserID() == userID)
            return player;
    }
    return shared_ptr<Player>();
}
```

no hash map — just a linear scan. same for `playerFromCharacter`, which compares `player->getCharacter()` against the passed instance.

```cpp
// read the players vector the same way you'd read Instance children
// Players::players is a copy_on_write_ptr, same layout as Instance::children
// dereference to get shared_ptr<vector<shared_ptr<Instance>>>
// each element is a shared_ptr<Player> — read the raw Instance* from the first pointer
```

---

## Local Player

`localPlayer` is a `shared_ptr<Player>`. it's null on the server, null in edit mode, and set to a real Player object on the client.

`createLocalPlayer` throws if called when one already exists. it creates the player, sets the userId, parents it to the Players service, then creates a StarterGear. it also does an early spawn location calculation in play-solo mode:

```cpp
shared_ptr<Instance> Players::createLocalPlayer(int userId, bool teleportedIn)
{
    if (localPlayer)
        throw std::runtime_error("Local player already exists");
    // ... creates Player, sets userId, parents it, creates StarterGear
    // in play-solo: calls doFirstSpawnLocationCalculation
}
```

`resetLocalPlayer` unlocks the parent lock, destroys the player, and resets the shared_ptr.

`getLocalPlayerDangerous()` is the reflection getter — it returns `localPlayer.get()` directly without incrementing the ref count. it's named "dangerous" because the raw pointer can dangle if nothing else holds the shared_ptr.

```cpp
uintptr_t getLocalPlayer(uintptr_t playersBase) {
    // localPlayer is shared_ptr<Player> — read the raw pointer from it
    uintptr_t sharedPtrStorage;
    ReadProcessMemory(hProcess, (LPVOID)(playersBase + Offsets::LocalPlayer), &sharedPtrStorage, sizeof(uintptr_t), nullptr);
    return sharedPtrStorage; // raw Player* (null on server/edit)
}
```

---

## Game Mode Detection

this is the most useful part of `Players` for context-sensitive code. the engine determines where it's running from three booleans:

```
clientIsPresent  — a Client service exists in the DataModel
serverIsPresent  — a Server service exists in the DataModel
localPlayer      — Players::findLocalPlayer returns non-null
```

from those three it derives the full mode table:

```
GameServer:   server only              serverIsPresent && !client
DPHYS_GameServer: same + dphys flag
Client:       client + localPlayer     client && localPlayer  (normal gameplay)
DPHYS_Client: same + dphys flag
WatchOnline:  client, no localPlayer   client && !localPlayer (spectating)
VisitSolo:    no client, localPlayer   !client && !server && localPlayer (solo play)
LocalPlay:    placeID <= 0             (when placeID overload used)
Edit:         nothing                  !client && !server && !localPlayer
```

the two helper functions are:

```cpp
frontendProcessing = serviceProvider exists && !serverIsPresent
backendProcessing  = serviceProvider exists && !clientIsPresent
```

in play-solo mode both are true simultaneously. `getGameMode` is inlined in the header so it compiles into call sites directly.

---

## onChildAdded — Player Join Flow

when a Player is parented to Players service, `onChildAdded` runs:

1. if a player with the same userId already exists, boot the old one (`setParent(NULL)`)
2. push to the `players` vector
3. notify FriendService
4. if backend (server or solo): assign test player name/id if userId == 0 and name is "Player", create PlayerGui, connect `scriptSecurityErrorSignal`, `killPlayerSignal`, `statsSignal`, `remoteFriendServiceSignal`
5. if not CloudEdit and spawn location not already calculated: assign team via TeamsService
6. fire `playerAddedEarlySignal` then `playerAddedSignal`
7. update `Http::playerCount`

the FirePlayerAddedAndPlayerRemovingOnClient DFFlag controls whether step 6 happens on both client and server or only backend. when the flag is off, `playerAdded` only fires on the server.

---

## onChildRemoving — Player Leave Flow

when a Player is removed:

1. notify FriendService
2. erase from `players` vector
3. fire `playerRemovingSignal` and `playerRemovingLateSignal` (conditional on DFFlag or backendProcessing)
4. update `Http::playerCount`

there's also `onDescendantRemoving` which locks the parent on any Player that's being removed while on the backend — this prevents the removed Player from being re-parented back in, which is an exploit protection.

---

## Chat

the chat history is a `std::list<ChatMessage>` capped by `GameSettings::singleton().chatHistory`. new messages are pushed to the front, old ones popped from the back.

`raiseChatMessageSignal` is the central routing point. it first tries to parse the message as a command:

```cpp
static bool ParseChatCommand(const ChatMessage& msg, bool isSelf, std::string* parsed_msg)
{
    if (msg.message[0] != '/') return false;
    // tokenize on " -_/,"
    // if first token (lowercased) == "sc": parse SafeChat message
}
```

only `/sc` commands are handled. if it's a `/sc` command, the message is replaced with the SafeChat text and the modified version is fired. otherwise if `localPlayer` has SuperSafeChat on, the message is dropped entirely. otherwise it fires `chatMessageSignal` and `playerChattedSignal`.

chat types:
```
CHAT_TYPE_ALL     — visible to everyone
CHAT_TYPE_TEAM    — visible to same team (non-neutral, same team color)
CHAT_TYPE_WHISPER — visible only to source and destination
CHAT_TYPE_GAME    — server-side game message, fires gameAnnounceSignal
```

team chat visibility check:
```cpp
case CHAT_TYPE_TEAM:
    return (player && source
            && !source->getNeutral() && !player->getNeutral()
            && source->getTeamColor() == player->getTeamColor());
```

so neutral players can't see team chat and can't send visible team chat.

whisper chat goes through GUID-based player identification in the bitstream — the source and destination are written as `{scope, index}` pairs from the GUID registry, not by userId. if `FilterInvalidWhisper` DFFlag is on and the destination GUID doesn't resolve to a valid player, the packet is dropped immediately.

---

## CharacterAutoLoads

`characterAutoSpawn` (exposed as `CharacterAutoLoads`). default `true`. controls whether players get a character automatically.

`getShouldAutoSpawnCharacter()` returns `!isCloudEdit(this) && characterAutoSpawn` — CloudEdit always suppresses auto-spawn regardless of the property.

when `characterAutoSpawn` is false and we're in visit/filtering mode, `loadLocalPlayerGuis` (connected to `gameLoadedSignal`) calls `localPlayer->rebuildGui()` to make sure GUIs still get created even without a character.

---

## Block/Unblock Flow

the block system has a client/server split:

**on the client:** `internalBlockUser` checks for a pending request, stores the callbacks in `clientBlockUserMap`, then fires `event_blockUserFromClient` (a remote event going client→server). the server receives it and calls `serverMakeBlockUserRequest` which POSTs to the API. when the API responds, the server fires `event_serverFinishedBlockUser` back to the client. the client receives it in `clientReceiveBlockUserFinished` and calls the stored callbacks.

**on the server (solo play or server-side call):** goes directly to `serverMakeBlockUserRequest`.

if a block request is already pending for a given pair of userids, subsequent calls error immediately without sending another request.

---

## Anti-Cheat Hash System

there are two hash systems:

**golden hashes** — a set of expected hash strings representing legitimate clients. set via `setGoldenHashes` (instance-level, three hashes for Windows/Mac/WPBeta) or `setGoldenHashes2` (static, a full set). `hashMatches` checks against all of them. on CloudEdit with the DFFlag on, it always returns true. the special string `"ios,ios"` also always passes.

```cpp
bool Players::hashMatches(const std::string& hash)
{
    // CloudEdit bypass
    // check goldenHashes set
    // check goldenHash / goldenHash2 / goldenHash3
    // "ios,ios" always passes
}
```

`canKickBecauseRunningInRealGameServer` is set to `true` the first time any golden hash is registered. this flag gates whether hash failures actually result in kicks.

**gold mem hashes** — `MemHashConfigs` is a `vector<vector<MemHash>>`. each `MemHash` has a `checkIdx`, an expected `value`, and a `failMask`. `checkGoldMemHashes` takes a vector of unsigned ints and tries each config, counting mismatches. it returns a bitmask of which checks failed on the best-matching config (fewest errors). this is used to detect memory tampering.

`onRemoteSysStats` is how the server gets told about a suspicious client. it logs the stat to `cheatingPlayers[userId]` (a set per userid — duplicates are ignored). if `canKickBecauseRunningInRealGameServer` is true and the kick was desired, it calls `disconnectPlayer`.

---

## Offsets

```cpp
namespace Offsets {
    namespace Players {
        uintptr_t PlayersVec          = 0x???;  // copy_on_write_ptr<Instances> (same layout as Instance::children)
        uintptr_t RakPeer             = 0x???;  // ConcurrentRakPeer* (null = no network)
        uintptr_t MaxPlayers          = 0x???;  // int (default 12)
        uintptr_t PreferredPlayers    = 0x???;  // int
        uintptr_t LocalPlayer         = 0x???;  // shared_ptr<Player> (null on server/edit)
        uintptr_t ChatOption          = 0x???;  // int (0=Classic, 1=Bubble, 2=ClassicAndBubble)
        uintptr_t CharacterAutoSpawn  = 0x???;  // bool (default true)
        uintptr_t TestPlayerNameId    = 0x???;  // int (counter)
        uintptr_t TestPlayerUserId    = 0x???;  // int (counter, goes negative)
        uintptr_t ChatHistory         = 0x???;  // std::list<ChatMessage>
        uintptr_t AbuseReporter       = 0x???;  // scoped_ptr<AbuseReporter> (null = no url set)
        uintptr_t GoldenHash          = 0x???;  // std::string
        uintptr_t GoldenHash2         = 0x???;  // std::string
        uintptr_t GoldenHash3         = 0x???;  // std::string
        uintptr_t ChatFilterUrl       = 0x???;  // std::string
        uintptr_t SysStatsUrl         = 0x???;  // std::string

        // statics (module base)
        uintptr_t CanKickBecauseRealGameServer = 0x???;  // bool
        uintptr_t GoldenHashes                = 0x???;  // std::set<std::string>
        uintptr_t GoldMemHashes               = 0x???;  // MemHashConfigs
    }
}
```


---

*more stuff coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
