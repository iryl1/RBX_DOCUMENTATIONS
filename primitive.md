# Roblox Primitive Docs

`Primitive` is the physics engine's representation of a part. it's a completely separate object from `PartInstance` — `PartInstance` is the scripting/datamodel side, `Primitive` is what the physics world actually simulates. every physical part has one. understanding this split is pretty important if you're trying to read physics state from memory.

if this helps you out, drop a ⭐ or come hang: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)

---

## What this covers

- the PartInstance / Primitive split — what lives where
- fields — full layout
- geometry types and how geometry is stored
- body — position, velocity, mass
- anchoring and dragging — the fixed state and why they're combined
- surface types and surface data — the lazy allocation
- joints and contacts — the EdgeList system and how to iterate them
- fuzzy extents — the cached AABB
- network ownership fields
- size, sortSize, and the sizeMultiplier system
- `clipToSafeSize` — the actual limits
- touch reporting — what triggers Touched/TouchEnded
- **how to read/write all of this yourself via offsets**

---

## The PartInstance / Primitive Split

`PartInstance` owns a `Primitive` via a pointer. the instance handles scripting, replication, and the datamodel tree. the Primitive handles collision geometry, mass, joints, contacts, and physics simulation. they communicate through an `IMoving` interface — `PartInstance` implements `IMoving` and registers itself as the Primitive's owner via `setOwner`.

most physics properties you set on a `Part` in Lua go through `PartInstance` → `Primitive`. if you want to read or write physics state from memory, you go through the Primitive, not the Instance.

---

## Fields

```
world               — World* (null if not in simulation)
geometry            — Geometry* (owns the shape, allocated by newGeometry)
body                — Body* (owns position, velocity, mass, coordinate frame)
myOwner             — IMoving* (the PartInstance that owns this Primitive)

contacts            — EdgeList (active contact edges)
joints              — EdgeList (active joint edges)

networkOwner        — SystemAddress
guid                — Guid (used for spanning tree identification)

sortSize            — unsigned int (cached, 0 = needs recalculation)
worldIndex          — int (index in World's primitive list, -1 = not in world)

fuzzyExtents        — Extents (cached AABB)
fuzzyExtentsStateId — unsigned int (invalidation token)

specificGravity     — float
jointK              — float (cached joint stiffness)
jointKDirty         — bool

friction            — float (default 0.0)
elasticity          — float (default 0.75)

customPhysicalProperties — boost::flyweight<PhysicalProperties>
material            — CompactEnum<PartMaterial, uint16_t>

dragging            — bool
anchoredProperty    — bool
preventCollide      — bool
networkIsSleeping   — bool

networkOwnershipRule — CompactEnum<NetworkOwnership, uint8_t>
engineType           — CompactEnum<EngineType, uint8_t>
sizeMultiplier       — CompactEnum<SizeMultiplier, uint8_t>

surfaceType[6]      — CompactEnum<SurfaceType, uint8_t>[6]
surfaceData         — SurfaceData* (null if all surfaces are default)
```

a few things worth noting about the layout. several fields use `CompactEnum<T, uint8_t>` or `uint16_t` which means they're smaller than a regular int in memory — 1 or 2 bytes. `material` is `uint16_t`, `networkOwnershipRule`, `engineType`, and `sizeMultiplier` are all `uint8_t`. the six surface types are packed as six consecutive bytes.

---

## Geometry

the `geometry` pointer owns a heap-allocated geometry object. the type is determined at construction and can be changed via `resetGeometryType` while outside the world. the full list:

```
GEOMETRY_BLOCK          → Block
GEOMETRY_BALL           → Ball
GEOMETRY_CYLINDER       → Cylinder
GEOMETRY_WEDGE          → WedgePoly
GEOMETRY_PRISM          → PrismPoly
GEOMETRY_PYRAMID        → PyramidPoly
GEOMETRY_PARALLELRAMP   → ParallelRampPoly
GEOMETRY_RIGHTANGLERAMP → RightAngleRampPoly
GEOMETRY_CORNERWEDGE    → CornerWedgePoly
GEOMETRY_MEGACLUSTER    → MegaClusterPoly (takes `this` in constructor)
GEOMETRY_SMOOTHCLUSTER  → SmoothClusterGeometry (takes `this` in constructor)
GEOMETRY_TRI_MESH       → TriangleMesh
```

`GEOMETRY_BLOCK` is the default fallback. `resetGeometryType` deletes the old geometry, creates the new one, copies the size over, and notifies the world. it asserts that there are no auto-joints when you do this.

size is stored on the geometry object, not directly on the Primitive:
```cpp
const Vector3& getSize() const { return geometry->getSize(); }
```

so reading size means: read `geometry` pointer, then read size from the geometry object at whatever offset the vtable puts it.

---

## Size Limits

`setSize` passes through `clipToSafeSize` before writing:

```cpp
Vector3 Primitive::clipToSafeSize(const Vector3& newSize)
{
    static const float maxVolume = 64.0e6f * 64.0e6f;
    Vector3 safeSize = newSize
        .min(Vector3(2048.0f, 2048.0f, 2048.0f))
        .max(Vector3::zero());

    if ((safeSize.x * safeSize.y * safeSize.z) > maxVolume)
        safeSize.y = floorf(1.0e6f / (safeSize.x * safeSize.z));

    return safeSize;
}
```

max per-axis is 2048 studs. the volume cap kicks in after that and adjusts Y to keep total volume under the limit. writing directly to the geometry size offset bypasses this.

when `setSize` is called while in-world, it notifies the world of both extents and geometry changes, and marks `jointKDirty` and `fuzzyExtentsStateId` as stale.

---

## Body — Position, Velocity, Mass

`body` is a heap-allocated `Body` object. it owns the coordinate frame, velocity (linear + angular), mass, and inertia tensor. most physics reads go through here:

```cpp
const CoordinateFrame& getCoordinateFrame() const {
    return getConstBody()->getPvSafe().position;
}
```

`getPvSafe()` acquires a read lock. `getPvUnsafe()` / `getCoordinateFrameUnsafe()` skip it — the comment says the calling thread must hold the writer lock.

`setPV` has quite a bit of logic inside it — it updates the bullet collision object, checks if this primitive is the assembly root, notifies the world's contact manager if it moved, and tickles the primitive to wake it up:

```cpp
void Primitive::setPV(const PV& newPv)
{
    // updates bullet object
    // if assemblyRoot: visits all primitives in assembly for contact manager
    //                  tickles primitive if not fixed
    // if not in assembly: notifies owner (PartInstance) via notifyMoved()
}
```

setting CFrame while dragging also invalidates bullet contact cache on all active contacts — stale contact positions cause bad collision responses.

```cpp
uintptr_t getBodyPtr(uintptr_t primitiveBase) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(primitiveBase + Offsets::Body), &ptr, sizeof(uintptr_t), nullptr);
    return ptr;
}

// then read CoordinateFrame from body at body + Offsets::Body::CoordinateFrame
```

---

## Fixed State — Anchoring and Dragging

`anchoredProperty` and `dragging` are separate bools but treated as a combined "fixed" state. the engine never checks them individually for fixedness — it always calls `requestFixed()`:

```cpp
bool requestFixed() const { return (dragging || anchoredProperty); }
```

and the only function that writes them is `setFixed`:

```cpp
void Primitive::setFixed(bool newAnchoredProperty, bool newDragging)
{
    bool wasFixed = (anchoredProperty || dragging);
    bool willBeFixed = (newAnchoredProperty || newDragging);

    bool update = (wasFixed != willBeFixed);

    if (update && world) world->onPrimitiveFixedChanging(this);
    anchoredProperty = newAnchoredProperty;
    dragging = newDragging;
    if (update && world) world->onPrimitiveFixedChanged(this);
}
```

the world only gets notified when the combined fixed state *transitions* — going from `(anchored=true, dragging=false)` to `(anchored=false, dragging=true)` doesn't notify the world because it was fixed before and is still fixed. only the transition fixed→unfixed or unfixed→fixed triggers the world callbacks.

`preventCollide` is separate — it's for the dragging-no-collide behavior. `getCanCollide()` returns `!dragging && !preventCollide`.

```cpp
bool getAnchoredProperty(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::AnchoredProperty), &value, sizeof(bool), nullptr);
    return value;
}

bool getDragging(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Dragging), &value, sizeof(bool), nullptr);
    return value;
}
```

---

## Surface Types and Surface Data

surface types are stored in two places. the type enum for each face (which joint connector type it uses) is in `surfaceType[6]` — six consecutive `uint8_t` values, one per `NormalId` face.

the surface data (for motor/hinge parameters) is lazily allocated:

```cpp
void Primitive::setSurfaceData(NormalId id, const SurfaceData& newSurfaceData)
{
    if (!surfaceData)
    {
        if (newSurfaceData.isEmpty())
            return;                     // don't allocate for empty data
        surfaceData = new SurfaceData[6];
    }
    surfaceData[id] = newSurfaceData;
}
```

`surfaceData` is null until something sets a non-empty surface data on any face. if it's null, all six faces return `SurfaceData::empty()`. reading the pointer and checking for null before dereferencing is required.

```cpp
uintptr_t getSurfaceData(uintptr_t base, int faceId /* 0-5 */) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::SurfaceDataPtr), &ptr, sizeof(uintptr_t), nullptr);
    if (!ptr) return 0; // null = all default
    return ptr + faceId * sizeof(SurfaceData);
}
```

---

## Joints and Contacts — EdgeList

joints and contacts are stored in separate `EdgeList` objects. each `EdgeList` is a `vector<Entry>` where each entry holds an `Edge*` and a `Primitive*` (the other primitive on that edge). this is a flat array — not a linked list.

```cpp
struct Entry {
    Edge* edge;
    Primitive* other;
};
```

removal uses swap-with-back: the removed entry gets replaced by the last entry, then the last entry is popped. this means **order is not preserved** in the joint/contact lists.

to iterate joints:

```cpp
// joints: read the EdgeList at Offsets::Joints
// EdgeList has: Primitive* owner, vector<Entry> list
// vector<Entry>: [ptr to data][size][capacity]
// each Entry is: Edge* (8 bytes) + Primitive* (8 bytes) = 16 bytes per entry

uintptr_t getJointsVector(uintptr_t base) {
    return base + Offsets::Joints + sizeof(uintptr_t); // skip owner ptr
}
```

`getNextEdge` iterates joints first then contacts — all joints come before any contacts in edge traversal order:

```cpp
Edge* Primitive::getNextEdge(Edge* e) const
{
    if (e->getEdgeType() == Edge::JOINT) {
        if (Edge* next = joints.getNext(this, e))
            return next;
        else
            return contacts.getFirst(); // fall through to contacts
    }
    else {
        return contacts.getNext(this, e);
    }
}
```

when looking up a joint between two specific primitives, the engine searches the shorter list first (`leastJoints`) and checks the `other` pointer for a match. same for contacts.

---

## Fuzzy Extents

`fuzzyExtents` is a cached axis-aligned bounding box, slightly expanded by `Tolerance::maxOverlapOrGap()` for the broadphase. it's invalidated by setting `fuzzyExtentsStateId = fuzzyExtentsReset()` (-2) which is guaranteed to be out of sync with the body's state index:

```cpp
const Extents& Primitive::getFastFuzzyExtents()
{
    if (fuzzyExtentsStateId != getBody()->getStateIndex()) {
        fuzzyExtents = computeFuzzyExtents();
        fuzzyExtentsStateId = getBody()->getStateIndex();
    }
    return fuzzyExtents;
}
```

`computeFuzzyExtents` uses `geometry->getCenterToCorner()` with the body's current rotation, so it accounts for orientation — it's a world-space AABB. reading `fuzzyExtents` directly without the state check can give you a stale box if the primitive has moved since it was last computed.

---

## SizeMultiplier and SortSize

`sizeMultiplier` is an enum that overweights certain special parts in the spanning tree so they become preferred roots:

```
DEFAULT_SIZE → 1
TORSO_SIZE   → 5
ROOT_SIZE    → 10
SEAT_SIZE    → 20
```

`sortSize` is computed from planar size × multiplier and used for joint sorting — heavier/larger parts become spanning tree roots. it caches as 0 when stale:

```cpp
void Primitive::calculateSortSize()
{
    unsigned int planarInt = Math::iFloor(getPlanarSize() * 50.0f);
    unsigned int multiplier = getSizeMultiplier();
    sortSize = planarInt * multiplier + 1; // 0 is reserved for uninitialized
}
```

`sizeMultiplier` can only be changed while `world` is null — there's an assert that crashes in debug if you try to change it while in-world.

---

## Touch Reporting

`Touched`/`TouchEnded` events (via `onNewOverlap`/`onStopOverlap`) only fire under specific conditions:

```cpp
if (  touchReporting->getOwner()->reportTouches()
   && (touchReportingAssembly->getAssemblyIsMovingState()
      || touchOtherAssembly->getAssemblyIsMovingState()
      || (DFFlag::FixTouchEndedReporting
          && (touchReporting->getDragging() || touchOther->getDragging()))))
```

both the reporting primitive's owner has to have `reportTouches()` true, AND at least one of the two assemblies has to be in a moving state (or one of them is being dragged if the fix flag is on). a contact between two completely sleeping/anchored parts won't fire touch events.

`reportOverlap` is called symmetrically for both primitives, so both parts get the event.

---

## Network Ownership

```
networkOwner         — SystemAddress (who controls this primitive's physics)
networkOwnershipRule — NetworkOwnership_Auto (0) or NetworkOwnership_Manual (1)
networkIsSleeping    — bool, whether the network owner has put this to sleep
```

`setNetworkIsSleeping` notifies the owner (`PartInstance`) via `onNetworkIsSleepingChanged` when the value changes.

```cpp
uint8_t getNetworkOwnershipRule(uintptr_t base) {
    uint8_t value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::NetworkOwnershipRule), &value, sizeof(uint8_t), nullptr);
    return value; // 0 = Auto, 1 = Manual
}
```

---

## Offsets

```cpp
namespace Offsets {
    namespace Primitive {
        uintptr_t World               = 0x???;  // World* (null = not in simulation)
        uintptr_t Geometry            = 0x???;  // Geometry* (heap, vtable determines type)
        uintptr_t Body                = 0x???;  // Body* (heap, owns CFrame/velocity/mass)
        uintptr_t MyOwner             = 0x???;  // IMoving* (PartInstance)
        uintptr_t Contacts            = 0x???;  // EdgeList (owner ptr + vector<Entry>)
        uintptr_t Joints              = 0x???;  // EdgeList (owner ptr + vector<Entry>)
        uintptr_t NetworkOwner        = 0x???;  // SystemAddress
        uintptr_t Guid                = 0x???;  // Guid
        uintptr_t SortSize            = 0x???;  // unsigned int (0 = stale)
        uintptr_t WorldIndex          = 0x???;  // int (-1 = not in world)
        uintptr_t FuzzyExtents        = 0x???;  // Extents (2x Vector3 = 24 bytes)
        uintptr_t FuzzyExtentsStateId = 0x???;  // unsigned int
        uintptr_t SpecificGravity     = 0x???;  // float
        uintptr_t JointK              = 0x???;  // float (cached)
        uintptr_t JointKDirty         = 0x???;  // bool
        uintptr_t Friction            = 0x???;  // float (default 0.0)
        uintptr_t Elasticity          = 0x???;  // float (default 0.75)
        uintptr_t CustomPhysProps     = 0x???;  // boost::flyweight<PhysicalProperties>
        uintptr_t Material            = 0x???;  // uint16_t (CompactEnum<PartMaterial>)
        uintptr_t Dragging            = 0x???;  // bool
        uintptr_t AnchoredProperty    = 0x???;  // bool
        uintptr_t PreventCollide      = 0x???;  // bool
        uintptr_t NetworkIsSleeping   = 0x???;  // bool
        uintptr_t NetworkOwnershipRule= 0x???;  // uint8_t (0=Auto, 1=Manual)
        uintptr_t EngineType          = 0x???;  // uint8_t (0=Dynamics, 1=Humanoid)
        uintptr_t SizeMultiplier      = 0x???;  // uint8_t (0=Default,1=Torso,2=Root,3=Seat)
        uintptr_t SurfaceType         = 0x???;  // uint8_t[6] (six consecutive bytes)
        uintptr_t SurfaceDataPtr      = 0x???;  // SurfaceData* (null = all default)
    }
}
```

placeholders — dump them yourself, they move between builds.

---

*more stuff coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
