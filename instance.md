# Roblox Instance System Docs

`Instance` is the base class everything inherits from. most of this you never need to think about until you're reading memory, and then suddenly it matters a lot — especially how the children list is laid out, what `OnDemandInstance` is, and what actually happens when you call `setParent`.


---

## What this covers

- fields — what's stored on every Instance
- the children list — layout in memory and how to iterate it
- `OnDemandInstance` — what it is and why signals aren't always allocated
- name — the flyweight string and the 100 char cap
- parent locking — the flags and what triggers them
- `setParent` — the full chain of what happens internally
- `destroy` vs `remove` — they're different
- `raisePropertyChanged` — the signal chain and the thread lock check
- archivable and robloxLocked
- **how to read/write all of this yourself via offsets**

---

## Fields

```
archivable      — bool, whether this instance is saved/cloned
isParentLocked  — bool, whether setParent will throw
robloxLocked    — bool, plugin-only flag
isSettingParent — bool, reentrancy guard for setParent
name            — boost::flyweight<std::string>
children        — copy_on_write_ptr<vector<shared_ptr<Instance>>>
parent          — Instance* (raw pointer)
onDemandPtr     — scoped_ptr<OnDemandInstance>, null until first signal connection
```

the important ones for memory reading are `name`, `children`, and `parent`. the rest are mostly single bools you can read individually.

---

## Children List

this is the one that actually requires some work to read. `children` is a `copy_on_write_ptr` which internally stores a pointer to a `shared_ptr<vector<shared_ptr<Instance>>>`. the layout in memory is:

```
[Instance base]
  + Offsets::ChildrenStart  → pointer to the shared_ptr control block / vector wrapper
                              dereference this to get a pointer to the actual vector
  + Offsets::ChildrenEnd    → (inside the vector) pointer to end of the element array
```

the vector itself is a `std::vector<shared_ptr<Instance>>`, so each element is a `shared_ptr` — two pointers wide (16 bytes on x64): `[px][pn]` where `px` is the raw `Instance*` you want.

here's how to iterate it (credit to user example):

```cpp
std::vector<Instance> Instance::GetChildren() const
{
    if (!address || !memory->IsValidAddress(address)) return {};

    std::uint64_t start = memory->Read<std::uint64_t>(address + Offsets::Instance::ChildrenStart);
    if (!memory->IsValidAddress(start)) return {};

    std::uint64_t end = memory->Read<std::uint64_t>(start + Offsets::Instance::ChildrenEnd);
    if (!memory->IsValidAddress(end) || end == 0) return {};

    std::vector<Instance> children;
    children.reserve(32);

    int count = 0;
    for (std::uint64_t instance = memory->Read<std::uint64_t>(start);
         instance != end && count < 100000;
         instance += sizeof(std::shared_ptr<void*>), count++)
    {
        if (!memory->IsValidAddress(instance)) break;
        std::uint64_t child_addr = memory->Read<std::uint64_t>(instance);
        if (memory->IsValidAddress(child_addr))
            children.emplace_back(child_addr);
    }
    return children;
}
```

the cap of 100000 is a sanity guard. in the engine itself, removing a child from a parent with >20 children uses a fast-remove (swap with back + pop) instead of an ordered erase:

```cpp
if (c->size() > 20)
{
    *iter = c->back();
    c->pop_back();
}
else
{
    c->erase(iter); // preserve order for small lists
}
```

so if you're reading children and order matters, be aware that large lists may not be in insertion order after removals.

one more thing: when the last child is removed, the engine resets the children pointer entirely rather than keeping an empty vector:

```cpp
if (c->size() == 1)
{
    c->clear();
    oldParent->children.reset(); // never keep an empty list around
}
```

so a null `children` pointer means zero children, not an uninitialized state. check for null before trying to dereference.

---

## Parent

`parent` is a raw `Instance*` stored directly on the object. reading it is straightforward:

```cpp
uintptr_t getParent(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Parent), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // raw Instance*, null if no parent
}
```

`getRootAncestor` just walks parent pointers until it hits null — no magic there.

---

## Name

name is stored as a `boost::flyweight<std::string>`. flyweight is an interning system — identical strings share the same underlying storage. what's actually stored on the instance is a handle/pointer into the flyweight pool, not the string itself.

the engine caps name length at 100 characters in the setter:

```cpp
void Instance::setName(const std::string& value)
{
    if (name.get() != value)
    {
        if (value.size() > 100)
            name = value.substr(0, 100);
        else
            name = value;
        this->raisePropertyChanged(desc_Name);
        checkParentWaitingForChildren(); // wakes up WaitForChild threads
    }
}
```

after setting the name it also calls `checkParentWaitingForChildren` — so if any Lua thread was blocked on `WaitForChild("SomeName")` and you rename an instance to match, it wakes up.

reading a flyweight from memory means reading through the flyweight handle to the interned string. the exact layout depends on the boost version but generally: `flyweight` stores a pointer to a shared flyweight value, which contains the `std::string`.

---

## OnDemandInstance

signals on `Instance` are not always allocated. the per-instance signals (ChildAdded, ChildRemoved, DescendantAdded, DescendantRemoving) live inside an `OnDemandInstance` object that only gets created the first time something tries to connect to or fire them:

```cpp
OnDemandInstance* Instance::onDemandWrite()
{
    OnDemandInstance* onDemand = onDemandPtr.get();
    if (onDemand == NULL)
    {
        onDemandPtr.reset(initOnDemand());
        onDemand = onDemandPtr.get();
    }
    return onDemand;
}
```

`onDemandRead()` just returns `onDemandPtr.get()` without creating it. the signal firing code checks this:

```cpp
void childAddedSignal(shared_ptr<Instance>& inst) {
    if (onDemandRead()) onDemandWrite()->childAddedSignal(inst);
}
```

so if nothing has ever connected to ChildAdded on an instance, `onDemandPtr` is null and the check short-circuits. no allocation, no firing.

the `onDemandPtr` is a `scoped_ptr` so if it's non-null there's a valid `OnDemandInstance` at that address. `ancestryChangedSignal` and `propertyChangedSignal` are different — they're stored directly on the `Instance` itself, not in the OnDemand block.

```cpp
uintptr_t getOnDemandPtr(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::OnDemandPtr), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // null if no signals ever connected
}
```

---

## Parent Locking

`isParentLocked` prevents `setParent` from working. anything with a locked parent throws if you try to reparent it:

```cpp
if (!ignoreLock && getIsParentLocked()) {
    throw std::runtime_error("The Parent property of X is locked...");
}
```

things with locked parents include all Services, the DataModel, Workspace, replicators, and player objects when removed. the lock can be bypassed internally via `setLockedParent` (which passes `ignoreLock = true`), but not from external code.

`setAndLockParent` locks first, tries to set, and unlocks again only if it fails — so on success the parent stays locked.

```cpp
bool getIsParentLocked(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::IsParentLocked), &value, sizeof(bool), nullptr);
    return value;
}

void setIsParentLocked(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::IsParentLocked), &value, sizeof(bool), nullptr);
}
```

---

## setParent — what actually happens

`setParentInternal` does a lot. in order:

1. no-op if `newParent == parent`
2. throws if parent is locked (unless `ignoreLock`)
3. throws if trying to parent to itself
4. throws if it would create a circular reference (`isAncestorOf` check)
5. checks `isSettingParent` reentrancy guard — if already mid-set, logs a warning and returns false
6. calls `verifySetParent`, `verifySetAncestor`, `verifyAddChild`, `verifyAddDescendant` — any of these can throw
7. removes from old parent's children list, using fast-remove if the list is >20 elements
8. adds to new parent's children list
9. fires `ChildRemoved` signals on old parent
10. fires anti-exploit `checkRbxCaller` (this is what detects certain dll injections)
11. fires `ChildAdded` signals on new parent
12. calls `checkParentWaitingForChildren` (wakes up WaitForChild)
13. calls `onAncestorChanged` recursively down through all descendants
14. calls `raiseChanged(propParent)`

the exploit detection at step 10 is notable — it calls `checkRbxCaller` and `detectDllByExceptionChainStack` specifically when `newParent == NULL` (removing from game). this is the game's detection layer for certain parent manipulation exploits.

---

## destroy vs remove

these are different and the difference matters:

**`remove()`** — sets parent to NULL, then recursively calls `remove()` on all children. non-destructive: the instance still exists and can be re-parented.

**`destroy()`** — sets parent to NULL (with lock), recursively calls `destroy()` on children, then disconnects all event listeners to break Lua reference cycles. after `destroy()` the instance is dead — it's no longer usable and any Lua references to it will be in a destroyed state.

from `destroy()`:

```cpp
std::for_each(
    getDescriptor().begin<Reflection::EventDescriptor>(),
    getDescriptor().end<Reflection::EventDescriptor>(),
    boost::bind(&Reflection::EventDescriptor::disconnectAll, _1, this)
);
```

that loop is specifically to prevent memory leaks from Lua closures holding references. `remove()` doesn't do this.

`destroyAllChildrenLua` (exposed as `ClearAllChildren`) also has a security check via `checkRbxCaller` that can result in a kick if called from the wrong context.

---

## raisePropertyChanged

the chain when a property changes:

```cpp
void Instance::raisePropertyChanged(const RBX::Reflection::PropertyDescriptor& descriptor)
{
    PropertyChanged event(...);
    this->onPropertyChanged(descriptor);           // virtual, subclass hook
    if (this->parent && DFFlag::LockViolationInstanceCrash)
        validateThreadAccess(this);                // crash if no DataModel write lock
    combinedSignal(PROPERTY_CHANGED, &data);       // combined signal
    propertyChangedSignal(&descriptor);            // Changed event
    if (getParent())
        getParent()->onChildChanged(this, event);  // propagates up the tree
}
```

the thread lock check only fires if `LockViolationInstanceCrash` DFFlag is on AND the instance has a parent. `validateThreadAccess` crashes the process if the calling thread doesn't hold the DataModel write lock — this is an anti-exploit measure.

`combinedSignal` fires first before `propertyChangedSignal`. `combinedSignal` is an optimization for listeners that want to monitor multiple signal types without allocating separate connections — you connect once and get everything.

---

## Archivable and RobloxLocked

`archivable` (default `true`) — controls whether the instance is included in save files and `Clone()`. the `writeXml` method returns `NULL` immediately if `!getIsArchivable()`, so non-archivable instances are silently skipped during serialization. `Clone()` also skips them in the `Cloner::createClones` pass.

`robloxLocked` (default `false`) — plugin-only. the setter has a `checkRbxCaller` that validates it's being called from legitimate code. reading/writing this via memory doesn't trigger that check.

```cpp
bool getIsArchivable(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Archivable), &value, sizeof(bool), nullptr);
    return value;
}

bool getRobloxLocked(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::RobloxLocked), &value, sizeof(bool), nullptr);
    return value;
}
```

---

## Offsets

```cpp
namespace Offsets {
    namespace Instance {
        uintptr_t Archivable      = 0x???;  // bool (default true)
        uintptr_t IsParentLocked  = 0x???;  // bool
        uintptr_t RobloxLocked    = 0x???;  // bool
        uintptr_t IsSettingParent = 0x???;  // bool (reentrancy guard)
        uintptr_t Name            = 0x???;  // boost::flyweight<std::string> — follow to interned string
        uintptr_t ChildrenStart   = 0x???;  // copy_on_write_ptr — dereference to get vector start
        uintptr_t ChildrenEnd     = 0x???;  // offset from ChildrenStart dereferenced to get vector end
        uintptr_t Parent          = 0x???;  // Instance* (raw, null = no parent)
        uintptr_t OnDemandPtr     = 0x???;  // scoped_ptr<OnDemandInstance> (null = no signals allocated)
    }
}
```


---

*more stuff coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
