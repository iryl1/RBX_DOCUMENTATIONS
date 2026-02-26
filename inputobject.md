# Roblox InputObject System Docs

`InputObject` is the engine custom input handler


---

## What this covers

- the fields that make up an InputObject
- UserInputType and UserInputState enums — full lists
- position and delta — how the gui inset affects what you read
- keyboard fields — keycodes, scancodes, mod codes, and the two systems
- which fields don't raise property changed events (and why)
- the isPublicEvent check — what makes an event "visible"
- **how to read/write all of this yourself via offsets**

---

## Fields

an InputObject has these fields:

```
inputType       — UserInputType enum, what kind of input this is
inputState      — UserInputState enum, begin/change/end/cancel
position        — Vector3, screen position for mouse/touch, axis values for gamepad
delta           — Vector3, movement delta since last event
keyCode         — SDL keycode for keyboard events
mod             — legacy modifier key bitfield (old keyboard system)
modifiedKey     — char, the actual typed character (old keyboard system)
modCodes        — unsigned int, modifier key bitfield (new keyboard system)
scanCode        — SDL scancode (new keyboard system)
inputText       — string, typed text for TextInput events
sourceInputType — the original inputType before any internal remapping
weakWorkspace   — weak pointer to the workspace, used for position/inset calculations
```

the last one is easy to miss — the InputObject holds a weak ref to the workspace so it can look up the GuiService inset and camera viewport when position is requested. it's set in every constructor that takes a `DataModel*`.

---

## UserInputType

```
MouseButton1    — left mouse button
MouseButton2    — right mouse button
MouseButton3    — middle mouse button
MouseWheel      — scroll wheel
MouseMovement   — mouse moved
Touch           — touchscreen
Keyboard        — key press
Focus           — window focus change
Accelerometer   — device accelerometer
Gyro            — device gyroscope
Gamepad1–8      — gamepad slots 1 through 8
TextInput       — text was typed (IME / soft keyboard)
None            — uninitialized / not set
```

there are also two internal types that don't appear in the enum registration — `TYPE_MOUSEIDLE` and `TYPE_MOUSEDELTA`. they get filtered out in `isPublicEvent()` and are only used internally by workspace tool processing. you'll see them if you read the raw `inputType` field but they'll never come through Lua.

---

## UserInputState

```
Begin   — input started (key down, button press, touch start)
Change  — input is ongoing and changed (mouse move, gamepad axis)
End     — input finished (key up, button release, touch end)
Cancel  — input was cancelled (touch interrupted, focus lost)
None    — uninitialized
```

---

## Position and the GUI Inset

`getPosition()` doesn't just return the raw position field. if the event is a screen position event (mouse or touch), it subtracts the GuiService's global inset:

```cpp
G3D::Vector3 InputObject::getPosition() const
{
    const Vector4 guiInset = getGuiInset();
    return isScreenPositionEvent()
        ? Vector3(position.x - guiInset.x, position.y - guiInset.y, position.z)
        : position;
}
```

the inset accounts for the topbar and any other UI chrome that shifts the usable screen area down. so `Position.Y = 0` from Lua actually corresponds to the top of the usable area, not the top of the window.

if you read the raw `position` field from memory you get the pre-inset screen coordinates. `getRawPosition()` is the internal method that does this — it just returns the field directly with no adjustment.

`getGuiInset()` works by locking the weak workspace pointer and asking the GuiService for `getGlobalGuiInset()`. if the workspace ref is dead it returns `Vector4::zero()` and position comes back unadjusted.

`get2DPosition()` is just `Vector2(position.x, position.y)` — drops the Z, no inset applied. used for raycasting from the camera.

```cpp
Vector3 getRawPosition(uintptr_t base) {
    float pos[3];
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Position), pos, sizeof(float) * 3, nullptr);
    return Vector3(pos[0], pos[1], pos[2]);
}

void setPosition(uintptr_t base, float x, float y, float z) {
    float pos[3] = { x, y, z };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Position), pos, sizeof(float) * 3, nullptr);
}

Vector3 getDelta(uintptr_t base) {
    float delta[3];
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Delta), delta, sizeof(float) * 3, nullptr);
    return Vector3(delta[0], delta[1], delta[2]);
}

void setDelta(uintptr_t base, float x, float y, float z) {
    float delta[3] = { x, y, z };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Delta), delta, sizeof(float) * 3, nullptr);
}
```

---

## Keyboard Fields — Two Systems

this is the messy part. there are two keyboard systems and the code routes between them based on a `UserInputService::IsUsingNewKeyboardEvents()` flag. each system has its own set of fields:

**old system:**
```
keyCode      — SDL keycode (SDLK_* values)
mod          — SDL_Keymod bitfield for modifier keys
modifiedKey  — char, the actual typed character after applying modifiers
```

**new system:**
```
keyCode      — same field, still used
scanCode     — SDL_Scancode (layout-independent key position)
modCodes     — unsigned int bitfield for modifiers (KMOD_LCTRL, KMOD_LALT, etc.)
inputText    — string, the typed text (used for TextInput events and IME)
```

the arrow key checks show this split clearly:

```cpp
bool InputObject::isLeftArrowKey() const
{
    if (UserInputService::IsUsingNewKeyboardEvents())
        return scanCode == SDL_SCANCODE_LEFT;
    return keyCode == SDLK_LEFT;
}
```

the new system uses scancode instead of keycode for navigation checks. scancodes are layout-independent — `SDL_SCANCODE_LEFT` is always the left arrow key regardless of keyboard layout, whereas `SDLK_LEFT` is the left arrow virtual key which could theoretically be remapped.

`isAltEvent()` and `isCtrlEvent()` branch the same way — check `modCodes` on the new system, `mod` on the old.

`setScanCode`, `setInputText`, and `setModCodes` are notable because they're commented out of raising property changed events:

```cpp
void InputObject::setScanCode(SDL_Scancode newScanCode)
{
    if (scanCode != newScanCode)
    {
        scanCode = newScanCode;
        //raisePropertyChanged(prop_KeyCode); // commented out
    }
}
```

same for `setInputText` and `setModCodes`. those three fields update silently — no event fires, nothing gets notified. the properties just exist on the struct.

```cpp
int getKeyCode(uintptr_t base) {
    int value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::KeyCode), &value, sizeof(int), nullptr);
    return value;
}

void setKeyCode(uintptr_t base, int value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::KeyCode), &value, sizeof(int), nullptr);
}

unsigned int getModCodes(uintptr_t base) {
    unsigned int value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::ModCodes), &value, sizeof(unsigned int), nullptr);
    return value;
}

void setModCodes(uintptr_t base, unsigned int value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ModCodes), &value, sizeof(unsigned int), nullptr);
}
```

---

## isPublicEvent

controls whether an event is surfaced to Lua:

```cpp
bool InputObject::isPublicEvent()
{
    if (inputType == InputObject::TYPE_MOUSEIDLE ||
        inputType == InputObject::TYPE_MOUSEDELTA ||
        sourceInputType != inputType)
    {
        return false;
    }
    return true;
}
```

two things filter an event out. first, `TYPE_MOUSEIDLE` and `TYPE_MOUSEDELTA` are always internal — they're used by workspace tool processing but never shown to scripts. second, if `sourceInputType != inputType` it means the event was remapped internally at some point (the `sourceInputType` is set at construction and never changed). remapped events don't go to Lua either.

so `sourceInputType` is basically a tamper marker. if you construct an InputObject with one type and then change `inputType` to something else, `isPublicEvent` will return false for it.

---

## Offsets

```cpp
namespace Offsets {
    // InputObject instance base
    uintptr_t InputType       = 0x???;  // int (UserInputType enum)
    uintptr_t InputState      = 0x???;  // int (UserInputState enum)
    uintptr_t Position        = 0x???;  // Vector3 (raw screen coords, pre-inset)
    uintptr_t Delta           = 0x???;  // Vector3
    uintptr_t KeyCode         = 0x???;  // int (SDL keycode)
    uintptr_t Mod             = 0x???;  // int (SDL_Keymod, old keyboard system)
    uintptr_t ModifiedKey     = 0x???;  // char (typed character, old keyboard system)
    uintptr_t ModCodes        = 0x???;  // unsigned int (modifier bitfield, new keyboard system)
    uintptr_t ScanCode        = 0x???;  // int (SDL_Scancode, new keyboard system)
    uintptr_t InputText       = 0x???;  // std::string (typed text / IME)
    uintptr_t SourceInputType = 0x???;  // int (UserInputType, original pre-remap type)
    uintptr_t WeakWorkspace   = 0x???;  // weak_ptr<Workspace>
}
```


---

*more stuff coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
