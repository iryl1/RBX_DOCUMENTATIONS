# Roblox Mouse System Docs

`Mouse` is what you get from `player:GetMouse()`. it's a thin wrapper over the input stream that does raycasting on demand. not much going on structurally but there's a few things that aren't obvious — mainly how `Hit` works, what `ViewSizeX/Y` actually measures, and the keyboard signal situation.


---

## What this covers

- fields — what's actually stored
- how `update` routes events into signals
- Hit, Origin, UnitRay — how the raycasting chain works
- Target and TargetFilter
- X, Y — where they come from and when they return -1
- ViewSizeX, ViewSizeY — why they're not just screen resolution
- Icon — the UserInputService stack interaction
- checkActive — what happens when the mouse dies
- **how to read/write all of this yourself via offsets**

---

## Fields

```
workspace    — raw Workspace*, set on construction, null when mouse is inactive
command      — raw MouseCommand*, the active tool, used for raycasting
icon         — TextureId, current custom cursor texture
lastEvent    — shared_ptr<InputObject>, snapshot of the most recent event
targetFilter — weak_ptr<Instance>, subtree excluded from raycasts
```

`lastEvent` is a cloned copy of the input, not the original:

```cpp
void Mouse::cacheInputObject(const shared_ptr<InputObject>& inputObject)
{
    lastEvent = Creatable<Instance>::create<InputObject>(*inputObject);
}
```

so reading `lastEvent` gives you a snapshot, not a live reference. the fields on it won't change until the next relevant event comes in.

---

## update — events to signals

all input comes through `update`. it switches on the input type and fires the right signal:

```cpp
void Mouse::update(const shared_ptr<InputObject>& inputObject)
{
    switch (inputObject->getUserInputType())
    {
    case TYPE_MOUSEMOVEMENT:
        cacheInputObject(inputObject);
        moveSignal();
        break;
    case TYPE_MOUSEIDLE:
        cacheInputObject(inputObject);
        idleSignal();
        break;
    case TYPE_MOUSEBUTTON1:
        cacheInputObject(inputObject);
        if (inputObject->isLeftMouseDownEvent())   button1DownSignal();
        else if (inputObject->isLeftMouseUpEvent()) button1UpSignal();
        break;
    case TYPE_MOUSEBUTTON2:
        cacheInputObject(inputObject);
        if (inputObject->isRightMouseDownEvent())   button2DownSignal();
        else if (inputObject->isRightMouseUpEvent()) button2UpSignal();
        break;
    case TYPE_MOUSEWHEEL:
        // no cacheInputObject here
        if (inputObject->isMouseWheelForward())    wheelForwardSignal();
        else if (inputObject->isMouseWheelBackward()) wheelBackwardSignal();
        break;
    case TYPE_KEYBOARD:
        // HACK ALERT! - Converting KeyCodes to ASCII this way only works for ASCII mapped keys!
        if (inputObject->isKeyDownEvent())
            keyDownSignal(std::string(1, inputObject->getKeyCode()));
        else if (inputObject->isKeyUpEvent())
            keyUpSignal(std::string(1, inputObject->getKeyCode()));
        break;
    }
}
```

two things to notice. wheel events skip `cacheInputObject` — so `lastEvent`, `X`, and `Y` don't update on scroll. the keyboard case casts the SDL keycode directly to a `char` — the source even has a comment calling it a hack that only works for ASCII-mapped keys. that's why `KeyDown`/`KeyUp` on `Mouse` are deprecated and don't work right for anything outside ASCII.

---

## Hit, Origin, UnitRay

none of these are cached — every access does work.

`getUnitRay` fires a ray from the camera through the current mouse position using `MouseCommand::getUnitMouseRay`:

```cpp
RBX::RbxRay Mouse::getUnitRay() const
{
    checkActive();
    RBX::RbxRay ray;
    if (lastEvent && lastEvent->getUserInputState() != INPUT_STATE_NONE)
        ray = MouseCommand::getUnitMouseRay(lastEvent, getWorkspace());
    return ray;
}
```

`getOrigin` takes that ray and wraps the origin into a CoordinateFrame pointed down the ray direction:

```cpp
CoordinateFrame Mouse::getOrigin() const
{
    checkActive();
    RBX::RbxRay ray;
    if (lastEvent && lastEvent->getUserInputState() != INPUT_STATE_NONE)
        ray = MouseCommand::getUnitMouseRay(lastEvent, getWorkspace());
    CoordinateFrame cf(ray.origin());
    cf.lookAt(ray.origin() + ray.direction());
    return cf;
}
```

`getHit` starts from `getOrigin()` and then casts into the world, writing the hit position into `cf.translation`:

```cpp
CoordinateFrame Mouse::getHit() const
{
    checkActive();
    CoordinateFrame cf(getOrigin());
    if (lastEvent)
    {
        FilterDescendents filter(getTargetFilter());
        MouseCommand::getPartByLocalCharacter(getWorkspace(), lastEvent, &filter, cf.translation);
    }
    return cf;
}
```

if nothing is hit, `cf.translation` stays as the camera origin. so `Hit.p` being the camera position is not a bug — it means the ray missed everything. the rotation on the returned frame always points down the ray regardless.

---

## Target and TargetFilter

`getTarget` returns the `PartInstance*` the ray hit, or null if nothing was hit or lastEvent has no state:

```cpp
PartInstance* Mouse::getTarget() const
{
    checkActive();
    FilterDescendents filter(getTargetFilter());
    return (lastEvent && lastEvent->getUserInputState() != INPUT_STATE_NONE) ?
        MouseCommand::getPart(getWorkspace(), lastEvent, &filter) : NULL;
}
```

`TargetFilter` is a `weak_ptr<Instance>`. the entire subtree under that instance is excluded from all three raycast properties — `Hit`, `Target`, and `TargetSurface`. setter fires a property change:

```cpp
void Mouse::setTargetFilterUnsafe(Instance* value)
{
    if (targetFilter.lock() != shared_from(value))
    {
        targetFilter = shared_from(value);
        raisePropertyChanged(desc_TargetFilter);
    }
}
```

since it's a weak ref, if the filter instance gets destroyed the lock returns null and raycasts just stop filtering — no error.

```cpp
uintptr_t getTargetFilter(uintptr_t base) {
    uintptr_t ptr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::TargetFilter), &ptr, sizeof(uintptr_t), nullptr);
    return ptr; // raw Instance* from weak_ptr internals
}
```

---

## X and Y

pulled directly from `lastEvent->getPosition()`, which applies the gui inset:

```cpp
int Mouse::getX() const
{
    checkActive();
    if (lastEvent)
        return lastEvent->getPosition().x;
    return -1;
}
```

returns `-1` if `lastEvent` is null — which happens before any mouse event has come in. and since wheel events don't update `lastEvent`, `X`/`Y` stay at the last known position if the user is only scrolling.

```cpp
int getX(uintptr_t base) {
    // read from lastEvent->position, not directly from Mouse
    uintptr_t lastEventPtr;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::LastEvent), &lastEventPtr, sizeof(uintptr_t), nullptr);
    float x;
    ReadProcessMemory(hProcess, (LPVOID)(lastEventPtr + InputOffsets::Position), &x, sizeof(float), nullptr);
    return (int)x;
}
```

---

## ViewSizeX and ViewSizeY

not the window size — they subtract the gui inset:

```cpp
int Mouse::getViewSizeX() const
{
    checkActive();
    if (GuiService* guiService = RBX::ServiceProvider::find<GuiService>(workspace))
        return guiService->getScreenResolution().x - guiService->getGlobalGuiInset().x;
    return 0;
}
```

same inset that `InputObject::getPosition()` subtracts. so `ViewSizeX/Y` gives you the usable game area below the topbar — not raw screen size. if you want the full resolution you'd need to get it from GuiService or the camera viewport directly.

---

## Icon

the cursor is a stack on `UserInputService`, not a single value. when you set a new icon the old one gets popped first, then the new one gets pushed:

```cpp
void Mouse::setIcon(const TextureId& value)
{
    if (icon != value)
    {
        if (UserInputService* userInputService = ServiceProvider::find<UserInputService>(workspace))
        {
            userInputService->popMouseIcon(icon);
            if (!value.toString().empty())
                userInputService->pushMouseIcon(value);
        }
        icon = value;
        raisePropertyChanged(desc_Mouse_Icon);
    }
}
```

setting the icon to an empty string pops the current one without pushing anything, restoring whatever was under it on the stack. `setWorkspace` also pops the icon off the old workspace's UserInputService when the workspace changes.

> **note:** writing to the `icon` offset directly skips the UserInputService stack. the Mouse thinks it has a new icon but the cursor won't change visually.

---

## checkActive

every getter calls this. `workspace` is set to null when the player leaves or the mouse is detached, and after that everything throws:

```cpp
void Mouse::checkActive() const
{
    if (!workspace)
    {
        RBXASSERT(0);
        throw std::runtime_error("This Mouse is no longer active");
    }
}
```

if you're reading `workspace` from memory and it's null, that's why everything else on the Mouse is dead too.

---

## Offsets

```cpp
namespace Offsets {
    // Mouse instance base
    uintptr_t Workspace     = 0x???;  // Workspace* (raw pointer, null = mouse inactive)
    uintptr_t Command       = 0x???;  // MouseCommand* (raw pointer)
    uintptr_t Icon          = 0x???;  // TextureId (std::string wrapper)
    uintptr_t LastEvent     = 0x???;  // shared_ptr<InputObject>
    uintptr_t TargetFilter  = 0x???;  // weak_ptr<Instance>
}
```


---

*more stuff coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
