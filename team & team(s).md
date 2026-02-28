# Roblox Team System Docs

---

## What this covers

- how `Teams` and `Team` objects are structured and stored
- how the `teams` list is kept in sync via child events
- how players get auto-assigned to teams
- how `getNumPlayersInTeam` works and what counts as "on a team"
- neutral vs. non-neutral players — what it means and how it's checked
- `isTeamGame` — when it returns true
- team color lookups — from color, from player, unused colors
- `getTeamColorForHumanoid` — how it walks the character tree
- `rebalanceTeams` — why it's a stub and what it was supposed to do
- **how to replicate this yourself**

> **note on `rebalanceTeams`:** the actual implementation is commented out in the source. the method is registered and callable but does nothing. it's marked deprecated in reflection. documented here for completeness.

---

## Data Model Structure

there are two classes: `Teams` (the service container) and `Team` (individual team objects that live inside it).

```
game
└── Teams  (service, one per game)
    ├── Team  (e.g. "Red Team")
    ├── Team  (e.g. "Blue Team")
    └── ...
```

`Teams` enforces this structure in both directions — `Team::askSetParent` rejects any parent that isn't a `Teams` instance, and `Teams::askAddChild` rejects any child that isn't a `Team` instance:

```cpp
// Team.cpp — only allows Teams as parent
bool Team::askSetParent(const Instance* parent) const {
    return Instance::fastDynamicCast<Teams>(parent) != NULL;
}

// Teams.h — only allows Team children
bool askAddChild(const Instance* child) const {
    return Instance::fastDynamicCast<Team>(child) != NULL;
}
```

so you can't accidentally put a `Team` somewhere weird or put a random `Instance` under `Teams`.

---

## Team Properties

each `Team` instance has four properties:

| Property | Type | Default | Deprecated? |
|---|---|---|---|
| `TeamColor` | `BrickColor` | `brickWhite` | no |
| `AutoAssignable` | `bool` | `true` | no |
| `Score` | `int` | `0` | yes |
| `AutoColorCharacters` | `bool` | `true` | yes |

the important ones are `TeamColor` and `AutoAssignable`. `Score` and `AutoColorCharacters` are deprecated and you shouldn't rely on them.

```cpp
Team::Team() :
    autoAssignable(true),
    score(0),
    autoColorCharacters(true)
{
    setName(sTeam);
    color = BrickColor::brickWhite();
}
```

all setters guard with a dirty check before firing `raisePropertyChanged`, so setting a property to the same value it already has is a no-op.

---

## The `teams` Cache

`Teams` maintains an internal `copy_on_write_ptr<Instances>` called `teams` that mirrors whatever `Team` children currently exist. it's kept up to date via `onChildAdded` and `onChildRemoving`:

```cpp
void Teams::onChildAdded(Instance* child) {
    if (Team* t = Instance::fastDynamicCast<Team>(child)) {
        boost::shared_ptr<Instances>& c = teams.write();
        c->push_back(shared_from(t));
    }
}

void Teams::onChildRemoving(Instance* child) {
    if (Team* t = Instance::fastDynamicCast<Team>(child)) {
        boost::shared_ptr<Instances>& c = teams.write();
        c->erase(std::find(c->begin(), c->end(), shared_from(t)));
    }
}
```

it ignores non-`Team` children entirely. `getTeams()` returns a read-only snapshot of this list. the `copy_on_write_ptr` means reads are cheap (no copy unless you write), and writes get a fresh copy so readers don't see partial state.

the cache is also initialized to a non-NULL empty list in the constructor:

```cpp
Teams::Teams() :
    teams(Instances())  // Initialize to a non-NULL value
{
    setName(sTeams);
}
```

this matters because the COW pointer could otherwise be null and crash on first read.

---

## Neutral vs. Non-Neutral Players

every `Player` has a `neutral` flag. a neutral player is not on any team. this is the core concept the whole system is built around:

- `getNeutral() == true` → player is not on a team, team color is meaningless
- `getNeutral() == false` → player is on a team, and `getTeamColor()` identifies which one

most team lookups skip neutral players entirely. for example `getNumPlayersInTeam` only counts a player if they're both non-neutral AND on the matching color:

```cpp
if (p->getNeutral() == false && p->getTeamColor() == color) result++;
```

---

## `isTeamGame()`

```cpp
bool Teams::isTeamGame() {
    if (Network::Players *players = ServiceProvider::find<Network::Players>(this)) {
        for (unsigned int n = 0; n < players->numChildren(); n++) {
            Network::Player* p = Instance::fastDynamicCast<Network::Player>(players->getChild(n));
            if (p == NULL) continue;
            if (p->getNeutral() == false)
                return true;
        }
    }
    return false;
}
```

returns `true` if **any** player in the game is non-neutral. it short-circuits on the first match — doesn't need to check everyone. if all players are neutral, it returns `false`.

this is a "is this game using teams at all" check, not "does this game have Team objects defined". a game can have `Team` objects parented under `Teams` and still return `false` from `isTeamGame()` if no player has been assigned to one yet.

---

## Auto-Assigning Players to Teams

```cpp
void Teams::assignNewPlayerToTeam(Network::Player *p) {
    BrickColor best = BrickColor::brickGreen();
    int best_count = 10000;
    bool found = false;

    for (unsigned int i = 0; i < this->numChildren(); i++) {
        Team* child = Instance::fastDynamicCast<Team>(this->getChild(i));
        if (child != NULL) {
            if (child->getAutoAssignable() == true) {
                int count = getNumPlayersInTeam(child->getTeamColor());
                if (count < best_count) {
                    best = child->getTeamColor();
                    best_count = count;
                    found = true;
                }
            }
        }
    }

    if (found) {
        p->setTeamColor(best);
        p->setNeutral(false);
    }
}
```

the algorithm is straightforward — find the auto-assignable team with the fewest players, put the new player there. ties go to whichever team appears first in child order (first `if (count < best_count)` doesn't update on equal). if no auto-assignable team exists, the player stays neutral.

`best_count` starts at `10000` which effectively means "any team beats the default". this is a sentinel, not a real player count.

note that `assignNewPlayerToTeam` is **not** called automatically when a player joins — it has to be called explicitly (e.g. from a script or the now-stubbed `rebalanceTeams`).

---

## `getNumPlayersInTeam(BrickColor color)`

walks all players, counts how many are non-neutral and on the given color:

```cpp
int Teams::getNumPlayersInTeam(BrickColor color) {
    int result = 0;
    Network::Players *players = ServiceProvider::find<Network::Players>(this);
    RBXASSERT(players);

    for (unsigned int n = 0; n < players->numChildren(); n++) {
        Network::Player* p = Instance::fastDynamicCast<Network::Player>(players->getChild(n));
        if (p == NULL) continue;
        if (p->getNeutral() == false && p->getTeamColor() == color) result++;
    }
    return result;
}
```

this is O(n) in the number of players, and `assignNewPlayerToTeam` calls it once per team, making assignment O(players × teams) overall. fine for normal game sizes.

---

## Team Color Lookups

### by color

```cpp
Team *Teams::getTeamFromTeamColor(BrickColor c) {
    for (unsigned int i = 0; i < numChildren(); i++) {
        Team *t = Instance::fastDynamicCast<Team>(getChild(i));
        if (t && t->getTeamColor() == c)
            return t;
    }
    return NULL;
}
```

linear scan through children, returns the first `Team` whose `TeamColor` matches. returns `NULL` if not found.

### by player

```cpp
Team *Teams::getTeamFromPlayer(Network::Player *p) {
    if (p->getNeutral())
        return NULL;
    return getTeamFromTeamColor(p->getTeamColor());
}
```

neutral players immediately return `NULL`. non-neutral players are looked up by their stored team color.

### checking if a team exists

```cpp
bool Teams::teamExists(BrickColor color) {
    return (getTeamFromTeamColor(color) != NULL);
}
```

thin wrapper around `getTeamFromTeamColor`.

---

## Getting an Unused Team Color

```cpp
BrickColor Teams::getUnusedTeamColor() {
    BrickColor::Colors vec(BrickColor::allColors());
    BrickColor::Colors::iterator iter;

    for (iter = vec.begin(); iter != vec.end(); iter++) {
        BrickColor c = *iter;
        for (unsigned int i = 0; i < numChildren(); i++) {
            if (Team *t = Instance::fastDynamicCast<Team>(getChild(i)))
                if (t->getTeamColor() == c)
                    vec.erase(iter);
        }
    }

    RBXASSERT(vec.size() > 0);
    return vec[rand() % vec.size()];
}
```

starts with all brick colors, erases any already in use by a child `Team`, then picks randomly from what's left.

> **note:** there's actually a subtle iterator invalidation bug here — `vec.erase(iter)` invalidates `iter` inside the outer loop but the outer loop continues using it. in practice this probably works most of the time because BrickColor lists are small and the loop tends to terminate before the UB causes visible issues, but it's not technically safe.

returns a random color from the remaining pool — so two calls in a row can return the same color if the pool has more than one entry.

---

## `getTeamColorForHumanoid(Humanoid *h)`

given a `Humanoid`, finds which player owns that character and returns their team color:

```cpp
G3D::Color3 Teams::getTeamColorForHumanoid(Humanoid *h) {
    Network::Players *players = ServiceProvider::find<Network::Players>(this);
    RBXASSERT(players);

    if (players) {
        for (unsigned int n = 0; n < players->numChildren(); n++) {
            if (Network::Player* p = Instance::fastDynamicCast<Network::Player>(players->getChild(n)))
                if (p->getNeutral() == false)
                    if (ModelInstance *m = p->getCharacter())
                        if (m->findFirstChildOfType<Humanoid>() == h)
                            return p->getTeamColor().color3();
        }
    }
    return G3D::Color3::white();
}
```

it walks all players, and for each non-neutral player it checks whether their character's first `Humanoid` child is the same pointer as `h`. if yes, return that player's team color as a `Color3`. if no match is found (neutral player, no character, wrong humanoid, etc.), returns white.

the pointer comparison `== h` means this only works if `h` is the actual `Humanoid` stored in the character model — no copies or wrappers.

---

## `rebalanceTeams()` — the Stub

```cpp
void Teams::rebalanceTeams() {
    // Do the elegant but slow thing - O(n^2)
    /*
    Network::Players *players = ServiceProvider::find<Network::Players>(this);
    RBXASSERT(players);

    for(unsigned int n = 0; n < players->numChildren(); n++) {
        Network::Player* p = Instance::fastDynamicCast<Network::Player>(players->getChild(n));
        p->setTeam(NULL);
        assignNewPlayerToTeam(p);
    }
    */
}
```

the body is completely commented out. the method is registered in reflection as deprecated, so it's callable from scripts but does nothing. the comment says O(n²) — iterating all players, and `assignNewPlayerToTeam` itself iterates players per team to count them. if it were active, it would clear every player's team then reassign them all using the least-populated team logic.

---


## Quick Reference

| Method | What it does |
|---|---|
| `isTeamGame()` | true if any player is non-neutral |
| `assignNewPlayerToTeam(p)` | puts player on least-populated auto-assignable team |
| `getNumPlayersInTeam(color)` | counts non-neutral players on a given team color |
| `teamExists(color)` | true if a Team child with that color exists |
| `getTeamFromTeamColor(c)` | returns Team object for a color, or NULL |
| `getTeamFromPlayer(p)` | returns Team for a player, or NULL if neutral |
| `getUnusedTeamColor()` | random BrickColor not already claimed by a Team child |
| `getTeamColorForHumanoid(h)` | returns team color of the player owning this humanoid, or white |
| `rebalanceTeams()` | does nothing (deprecated stub) |
| `getTeams()` | read-only snapshot of Team children |

---

## Things Worth Knowing

- team color is how the system identifies teams internally — not the team's name. two teams with the same color would break lookups.
- `neutral = true` means a player is effectively teamless regardless of what `teamColor` is set to.
- there's no automatic hook that calls `assignNewPlayerToTeam` on player join — you have to wire that up yourself.
- `getUnusedTeamColor` has an iterator invalidation bug in the source. don't rely on it in large color pools.
- `rebalanceTeams` is callable but fully stubbed out and deprecated.
- `Score` and `AutoColorCharacters` on `Team` are both deprecated — avoid using them.

---


> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
