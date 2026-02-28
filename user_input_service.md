# Roblox UserInputService System Docs

---

## What this covers

- platform detection — how the service knows what device it's on
- input device capability flags (`TouchEnabled`, `KeyboardEnabled`, etc.) — how they're set and what they mean
- the render step pipeline — the order everything processes each frame
- input event pipeline — how raw input becomes `InputBegan`/`InputChanged`/`InputEnded`
- core vs. script events — the two-tier event system and what the difference is
- `lastInputType` — how it's tracked and what it ignores
- keyboard state — how `newKeyState` works, `IsKeyDown`, `GetKeysPressed`
- `getModifiedKey` — shift/caps handling and its known limitations
- mouse icon stack — push/pop behavior
- `MouseBehavior` / `WrapMode` — how the lock modes work
- fake mouse events — the touch-to-mouse compatibility layer
- touch events and gesture processing
- character movement — how `MoveLocalCharacter` translates input into world direction
- camera manipulation — rotate, zoom, wrap, track
- motion sensors — accelerometer, gyroscope, gravity
- gamepad — connected map, supported key codes, `GamepadSupports`
- VR — `UserCFrame`, head recentering
- textbox focus tracking
- `showStatsBasedOnInputString` — the hidden debug cheat codes

---

## Platform Detection

platform is detected at compile time via preprocessor defines:

```cpp
UserInputService::Platform UserInputService::getPlatform() {
#if defined(RBX_PLATFORM_IOS)
    return PLATFORM_IOS;
#elif defined(__APPLE__) && !defined(RBX_PLATFORM_IOS)
    return PLATFORM_OSX;
#elif defined(RBX_PLATFORM_DURANGO)
    return PLATFORM_XBOXONE;
#elif defined(RBX_PLATFORM_UWP)
    return PLATFORM_UWP;
#elif defined(_WIN32) && !defined(RBX_PLATFORM_DURANGO) && !defined(RBX_PLATFORM_UWP)
    return PLATFORM_WINDOWS;
#elif defined(__ANDROID__)
    return PLATFORM_ANDROID;
#else
    return PLATFORM_NONE;
#endif
}
```

it's a static method so it doesn't need a service instance. the full platform enum has entries for Xbox One, PS3, PS4, Xbox 360, WiiU, NX, Ouya, AndroidTV, Chromecast, Linux, SteamOS, WebOS, DOS, BeOS, and UWP — mostly aspirational or unreleased targets at this point. the ones that are actually populated by the constructor logic are Windows, OSX, iOS, Android, and XboxOne.

`isTenFootInterface()` returns true for consoles and TV platforms (XboxOne, PS4, PS3, Xbox360, WiiU, Ouya, AndroidTV, SteamOS) — this is used to know if you're in a "lean back" 10-foot UI context.

---

## Input Device Capability Flags

the constructor sets input flags based on platform:

```cpp
switch (getPlatform()) {
    case PLATFORM_WINDOWS:
    case PLATFORM_OSX:
        lastInputType = InputObject::TYPE_MOUSEMOVEMENT;
        setMouseEnabled(true);
        setKeyboardEnabled(true);
        break;
    case PLATFORM_IOS:
    case PLATFORM_ANDROID:
        lastInputType = InputObject::TYPE_TOUCH;
        setTouchEnabled(true);
        break;
    case PLATFORM_XBOXONE:
        lastInputType = InputObject::TYPE_GAMEPAD1;
        setGamepadEnabled(true);
        break;
    default:
        lastInputType = InputObject::TYPE_NONE;
        break;
}
```

these flags are read-only from Lua (no setter is registered in reflection). the properties are `TouchEnabled`, `KeyboardEnabled`, `MouseEnabled`, `GamepadEnabled`, `AccelerometerEnabled`, `GyroscopeEnabled`. internally they do have setters — for example `setGamepadEnabled` gets called automatically when a gamepad connects or disconnects.

`getTouchEnabled()` has a special case: it also returns true if `isStudioEmulatingMobile` is set, even if `touchEnabled` itself is false:

```cpp
bool UserInputService::getTouchEnabled() const {
    return touchEnabled || isStudioEmulatingMobile;
}
```

`isStudioEmulatingMobile` is a static bool shared across all instances — toggling Studio's mobile emulation affects this globally.

---

## The Render Step Pipeline

every frame, `onRenderStep()` is called and runs everything in this order:

```
updateInputSignal()         → lets hardware devices flush their buffers into InputObjects
processInputObjects()       → fires Begin/Change/End events for all queued input
processKeyboardEvents()     → drains the keyboard event queue and updates newKeyState
processGestures()           → fires high-level gesture events (tap, pinch, swipe, etc.)
processMotionEvents()       → fires accelerometer/gyro/gravity events
processToolEvents()         → forwards tool-related input to the workspace
processCameraInternal()     → applies accumulated camera pan/zoom/wrap/track
processTextboxInternal()    → fires TextBoxFinishedEditing events
fireJumpRequestEvent        → fires JumpRequest if pending
cleanupCurrentTouches()     → removes ended touches from currentTouches
centerCursor()              → if MouseBehavior is LockCenter and conditions are met
```

the ordering matters — keyboard state is updated after `processInputObjects`, so anything that reads key state during input processing sees the state from the *previous* frame. if you're relying on `IsKeyDown` inside an input event, be aware of this.

---

## Input Event Pipeline

when a raw input event comes in (e.g. from SDL), it flows like this:

```
fireInputEvent(inputObject, nativeObject)
    → queues into beginEventsToProcess / changeEventsToProcess / endEventsToProcess
    → fireLegacyMouseEvent() [for touch compat]

onRenderStep → processInputObjects()
    → processInputVector(begin, INPUT_STATE_BEGIN)
    → processInputVector(change, INPUT_STATE_CHANGE)
    → processInputVector(end, INPUT_STATE_END)
        → dataModelProcessInput()
            → dataModel->processInputObject()   [checks GUI sinking]
            → signalInputEventOnService()       [fires InputBegan/Changed/Ended signals]
```

a key detail: **change events are deduplicated**. if you fire 10 mouse move events in one frame, only the last position matters — `fireInputEvent` accumulates delta values and replaces the position:

```cpp
storedChangedInput->setDelta(storedChangedInput->getDelta() + inputObject->getDelta());
storedChangedInput->setPosition(inputObject->getRawPosition());
```

so delta is *summed* across the frame, but position is the final position. for touch change events, the dedup is per event pair (not per type), since multiple fingers can be active at once.

begin and end events always get queued and always fire — no dedup on those.

---

## Core vs. Script Events — The Two-Tier System

`InputBegan`, `InputChanged`, `InputEnded` each actually have **two** signals internally: one for regular scripts and one for CoreScripts:

```cpp
rbx::signal<void(shared_ptr<Instance>, bool)> inputBeganEvent;
rbx::signal<void(shared_ptr<Instance>, bool)> coreInputBeganEvent;
```

which one you get when you listen to `InputBegan` depends on your security level:

```cpp
rbx::signal<void(shared_ptr<Instance>, bool)>* UserInputService::getInputBeganEvent(bool) {
    return RBX::Security::Context::current().hasPermission(RBX::Security::RobloxScript)
        ? &coreInputBeganEvent
        : &inputBeganEvent;
}
```

so CoreScripts get their own signal, and regular game scripts get the other. **the core signal always fires**. the regular signal only fires if `menuIsOpen` is false at the time of dispatch. same split applies to `InputChanged`, `InputEnded`, and all three touch events (`TouchStarted`, `TouchMoved`, `TouchEnded`).

when a menu is open (checked via `GuiService::getMenuOpen()`), game scripts stop receiving input events but CoreScripts continue to receive them uninterrupted.

---

## `lastInputType` — How It's Tracked

`lastInputType` is updated in `updateLastInputType`, which is called for every public input event:

```cpp
void UserInputService::updateLastInputType(const shared_ptr<InputObject>& inputObject) {
    if (lastInputType != inputObject->getUserInputType()) {
        // ignore tiny thumbstick / trigger movements
        if ( (inputObject->getKeyCode() == SDLK_GAMEPAD_THUMBSTICK1 || ...) &&
             inputObject->getRawPosition().magnitude() < 0.2 )
            return;

        lastInputType = inputObject->getUserInputType();
        lastInputTypeChangedSignal(lastInputType);
    }
}
```

the filter on thumbsticks and triggers (< 0.2 magnitude) means resting on a gamepad's analog stick won't pollute `lastInputType`. this prevents "last input was gamepad" from being set when a thumbstick is just jittering at rest.

`GetLastInputType()` is available to all scripts. `LastInputTypeChanged` fires whenever it changes.

---

## Keyboard State

the keyboard uses a `boost::unordered_map<RBX::KeyCode, shared_ptr<InputObject>>` called `newKeyState`. the map is populated lazily — the first time a key is pressed, an `InputObject` for it gets created and added. subsequent presses update the existing entry's state.

```cpp
void UserInputService::setKeyStateInternal(RBX::KeyCode keyCode, ModCode modCode, char modifiedKey, bool isDown) {
    if (newKeyState.find(keyCode) == newKeyState.end()) {
        newKeyState[keyCode] = create InputObject(TYPE_KEYBOARD, isDown ? STATE_BEGIN : STATE_END, ...);
    } else {
        newKeyState[keyCode]->setInputState(isDown ? STATE_BEGIN : STATE_END);
        newKeyState[keyCode]->mod = modCode;
        newKeyState[keyCode]->modifiedKey = modifiedKey;
    }
}
```

`IsKeyDown(keyCode)` checks if the entry exists and is in `INPUT_STATE_BEGIN`. a key that was never pressed doesn't exist in the map, so it returns false correctly without any special-casing.

`GetKeysPressed()` walks the whole map and returns all entries currently in `BEGIN` state. this is a snapshot — it won't change mid-frame.

`CapsLock` is tracked separately as a bool (`capsLocked`) that toggles on each key-down event for the caps lock key specifically.

keyboard events are queued through a separate `KeyboardEventsVector` with a dedicated mutex and processed in `processKeyboardEvents()` during the render step — they don't go through the normal `beginEventsToProcess` path.

### modifier keys

the `modPairs` vector maps keycodes to modifier flags and is platform-specific:

- **Windows:** Ctrl (left + right)
- **macOS:** Cmd/Meta (left + right)
- Both platforms add: Shift (left + right), CapsLock

`getCurrentModKey()` scans `modPairs` and returns the modifier flag for whatever modifier key is currently held down. `getCommandModCodes()` returns the platform command modifier (Ctrl on Windows, Cmd on macOS) for use in keyboard shortcut logic.

---

## `getModifiedKey` — Shift and Caps Handling

given a raw `KeyCode` and a `ModCode`, returns the actual character the user typed:

```cpp
bool isUpper = (shift && !caps) || (!shift && caps);  // XOR
```

uppercase is XOR between shift and caps lock — which is correct. letters a–z are handled cleanly. for symbols it handles shift+number to punctuation (1→!, 2→@, etc.), and shift+symbol to their shifted counterparts (;→:, ,→<, [→{, etc.).

> **note:** the comment in the source says "this is bad, doesn't work with non-standard keyboards" — the symbol mappings are hardcoded for US QWERTY and will silently produce wrong characters on other layouts.

modifier-only keys (shift, ctrl, caps, alt) return `0` — they don't produce a character.

---

## Mouse Icon Stack

the mouse cursor is managed as a **stack** of `TextureId` values. the top of the stack is the current cursor:

```cpp
TextureId UserInputService::getCurrentMouseIcon() {
    return (mouseIconStack.size() > 0) ? mouseIconStack.back() : TextureId();
}
void UserInputService::pushMouseIcon(TextureId newMouseIcon) { mouseIconStack.push_back(newMouseIcon); }
void UserInputService::popMouseIcon() {
    if (mouseIconStack.size() > 0) mouseIconStack.pop_back();
}
void UserInputService::popMouseIcon(TextureId mouseIconToRemove) {
    mouseIconStack.erase(std::remove(...mouseIconToRemove...), mouseIconStack.end());
}
```

the overloaded `popMouseIcon(TextureId)` removes **all instances** of a specific icon from anywhere in the stack, not just the top. this is useful for cleanup when you're not sure if your icon is still on top.

`mouseIconEnabled` (exposed as `MouseIconEnabled`) controls visibility independently of the stack — setting it to false fires `mouseIconEnabledEvent` which is what actually hides the cursor in the renderer.

there's also `OverrideMouseIconBehavior` (RobloxScript-only), which CoreScripts use to force-show or force-hide the cursor in menus regardless of what the game set.

---

## `MouseBehavior` and `WrapMode`

three mouse behavior modes:

| Enum | Lua Name | What happens |
|---|---|---|
| `MOUSETYPE_DEFAULT` | `Default` | normal free mouse |
| `MOUSETYPE_LOCKCENTER` | `LockCenter` | locks to screen center, re-centers every frame |
| `MOUSETYPE_LOCKCURRENT` | `LockCurrentPosition` | locks to wherever the cursor was when it was set |

`setMouseType` maps these to `WrapMode` internally:

```cpp
case MOUSETYPE_DEFAULT:
    setMouseWrapMode(WRAP_AUTO);
    break;
case MOUSETYPE_LOCKCENTER:
    userInput->centerCursor(); // immediately center on switch
    break;
```

`getMouseWrapMode()` is what the renderer actually reads. it translates the current `mouseType` into a wrap mode, but only if `canUseMouseLockCenter()` returns true:

```cpp
WrapMode UserInputService::getMouseWrapMode() const {
    switch (mouseType) {
        case MOUSETYPE_LOCKCENTER:
        case MOUSETYPE_LOCKCURRENT:
            if (canUseMouseLockCenter()) {
                return (mouseType == MOUSETYPE_LOCKCENTER) ? WRAP_NONEANDCENTER : WRAP_NONE;
            }
            break;
    }
    return wrapMode;
}
```

`canUseMouseLockCenter()` requires: a workspace exists, no modal GUI objects are open, and the profiler isn't capturing mouse input. if any of those conditions fail, the lock silently reverts to `WRAP_AUTO` (free mouse) until conditions are met again. `MOUSETYPE_LOCKCENTER` also re-centers the cursor every frame in `onRenderStep()` as long as `canUseMouseLockCenter()` is true.

---

## Fake Mouse Events — Touch Compatibility Layer

for legacy compatibility with GUI objects that only understand mouse events, touch events get translated into fake mouse events:

```cpp
void UserInputService::fireLegacyMouseEvent(const shared_ptr<InputObject>& inputObject, ...) {
    if (inputObject->isTouchEvent()) {
        InputObject::UserInputType fakeInputType;
        switch (inputObject->getUserInputState()) {
            case INPUT_STATE_BEGIN:
            case INPUT_STATE_END:
                fakeInputType = TYPE_MOUSEBUTTON1;
                break;
            case INPUT_STATE_CHANGE:
                fakeInputType = TYPE_MOUSEMOVEMENT;
                break;
        }
        // update fakeMouseEventsMap[fakeInputType] with touch position and fire it
    }
}
```

touch begin/end → MouseButton1 down/up. touch move → MouseMove. these fake events have their `SourceUserInputType` set to `TYPE_TOUCH` so you can still detect that the underlying input was touch if needed.

the fake event objects are pre-allocated in `onServiceProvider` and reused — positions/states get mutated in-place each time.

---

## Touch Events and Gestures

low-level touch events (`TouchStarted`, `TouchMoved`, `TouchEnded`) fire per finger. `getCurrentTouches()` returns all currently active touches — touches are added on `INPUT_STATE_BEGIN` and removed from `currentTouches` in `cleanupCurrentTouches()` after end events are processed.

high-level gesture events are buffered separately through `addGestureEventToProcess` (with its own mutex) and processed in `processGestures()`. before firing, touch positions get adjusted by the GUI inset offset (so positions are in screen space, not including any top bar). the processing order is: try CoreGui first, then PlayerGui, then fire the gesture event with `wasSunk` set accordingly.

gesture types and their signal parameters:

| Gesture | Signal | Args |
|---|---|---|
| `TouchTap` | `tapGestureEvent` | `touchPositions`, `gameProcessedEvent` |
| `TouchLongPress` | `longPressGestureEvent` | `touchPositions`, `state`, `gameProcessedEvent` |
| `TouchSwipe` | `swipeGestureEvent` | `swipeDirection`, `numberOfTouches`, `gameProcessedEvent` |
| `TouchPinch` | `pinchGestureEvent` | `touchPositions`, `scale`, `velocity`, `state`, `gameProcessedEvent` |
| `TouchRotate` | `rotateGestureEvent` | `touchPositions`, `rotation`, `velocity`, `state`, `gameProcessedEvent` |
| `TouchPan` | `panGestureEvent` | `touchPositions`, `totalTranslation`, `velocity`, `state`, `gameProcessedEvent` |

---

## Character Movement — How `MoveLocalCharacter` Works

`moveLocalCharacterLua(walkDir, maxWalkDelta)` is what Lua calls. it calls `moveLocalCharacterInternal` immediately and also caches `walkDirection` and `maxWalkDelta` on the service.

the actual movement translation is not trivial — it takes a 2D input direction and converts it into a world-space walk direction relative to the camera:

```cpp
// 1. get percent walk speed from analog magnitude
localHumanoid->setPercentWalkSpeed(walkDir.length() / walkDelta);

// 2. normalize the input direction
Vector2 movementVector = walkDir.direction();

// 3. find the angle between "north" (0,-1) and the input direction
float angle = -atan2(cross(movementVector, northDirection), dot(movementVector, northDirection));

// 4. get camera's look vector projected onto the XZ plane
Vector3 camLookProjected = project camera look onto XZ plane;
Vector2 camLook2d = Vector2(camLookProjected.x, camLookProjected.z);

// 5. rotate camLook2d by the angle
float xRotated = camLook2d.x * cosAngle - camLook2d.y * sinAngle;
float zRotated = camLook2d.x * sinAngle + camLook2d.y * cosAngle;

localHumanoid->setWalkDirection(Vector3(xRotated, 0, zRotated));
```

so when you call `MoveLocalCharacter(Vector2(1, 0), 1)`, the character moves in the direction that is 90° clockwise from the camera's look direction — i.e. to the right from the player's perspective. the input vector is camera-relative.

if `walkDir` is zero, walk speed is set to 1.0 and walk direction is `Vector3::zero()` — the character stops.

`jumpLocalCharacter(bool)` checks `localCharacterJumpEnabled` before setting the jump flag. `jumpOnceLocalCharacter` always fires the jump request regardless of the flag. `setLocalCharacterJumpEnabled` throws a runtime error if called from a non-local script.

---

## Camera Manipulation

camera operations are accumulated as deltas and flushed in `processCameraInternal()`:

| Function | What it accumulates |
|---|---|
| `rotateCamera(delta)` | `cameraPanDelta`, clamped to ±100 on X |
| `rotateCameraLua(delta)` | calls `panRadians`/`tiltRadians` immediately via DataModel task |
| `zoomCamera(zoom)` | `cameraZoom` (additive) |
| `zoomCameraLua(zoom)` | calls `camera->zoom()` immediately via DataModel task |
| `wrapCamera(wrap)` | `cameraMouseWrap` (additive) |
| `mouseTrackCamera(delta)` | `cameraMouseTrack` (replaces, not additive) |

the non-Lua versions buffer and apply during `processCameraInternal`. the Lua versions submit a task to the DataModel and execute immediately on the DataModel thread.

`processCameraInternal` bails out early if: `FFlag::UserAllCamerasInLua` is set and the camera has a client player (Lua camera control takes over), or if the camera type is `CUSTOM_CAMERA` and a local player exists (game's camera script is handling it). camera manipulation only applies to `LOCKED_CAMERA` type or when camera pan mode is `CAMERAPANMODE_CLASSIC`.

---

## Motion Sensors

three sensor types: accelerometer, gyroscope (rotation), gravity. each has its own `InputObject` that gets mutated in place when new data comes in. all three are protected by `MotionEventsMutex`.

```cpp
void UserInputService::fireRotationEvent(const RBX::Vector3& newRotation, const RBX::Vector4& quaternion) {
    // delta = new - old position
    gyroInput->setDelta(newRotation - gyroInput->getPosition());
    gyroInput->setPosition(newRotation);

    // quaternion → rotation matrix
    Quaternion quat(quaternion.w, quaternion.y, quaternion.z, quaternion.x);
    quat.normalize();
    quat.toRotationMatrix(rotationCFrame.rotation);

    shouldFireRotationEvent = true;
}
```

the quaternion argument uses a non-standard component order `(w, y, z, x)` — that's not a typo. `GetDeviceRotation()` returns a tuple of `(rotationInputObject, rotationCFrame)` — both the rate-of-rotation and the absolute orientation.

all three sensor event signals throw if you try to listen to them from a non-local script. gravity and rotation also log a warning if the device doesn't actually have the hardware (`accelerometerEnabled` / `gyroscopeEnabled` are false).

---

## Gamepad

gamepads 1–8 are tracked in `connectedGamepadsMap`. when any gamepad connects, `gamepadEnabled` is set to true. when all disconnect, it's set back to false:

```cpp
void UserInputService::setConnectedGamepad(InputObject::UserInputType gamepadType, bool isConnected) {
    connectedGamepadsMap[gamepadType] = isConnected;

    for (auto& pair : connectedGamepadsMap) {
        if (pair.second) {
            setGamepadEnabled(true);
            return;
        }
    }
    setGamepadEnabled(false);
}
```

`getSupportedGamepadKeyCodes(gamepadType)` fires `getSupportedGamepadKeyCodesSignal`, which is listened to by `SDLGameController` — the signal causes the controller to populate `supportedGamepadKeyCodes[gamepadType]`. the method then returns the now-populated array. so the supported key codes are fetched lazily on demand, not cached at connect time.

`gamepadSupports(gamepadType, keyCode)` just walks the supported key codes array doing a linear scan. `GamepadConnected` and `GamepadDisconnected` events are dispatched via a DataModel task (`safeFireGamepadConnected/Disconnected`) rather than directly — this ensures they fire on the write thread.

navigation gamepad tracking delegates entirely to `GamepadService`.

---

## VR

three `UserCFrame` types:

| Enum | Lua Name | What it is |
|---|---|---|
| `USERCFRAME_HEAD` | `Head` | head position/orientation |
| `USERCFRAME_LEFTHAND` | `LeftHand` | left controller |
| `USERCFRAME_RIGHTHAND` | `RightHand` | right controller |

`setUserCFrame(type, value)` fires `userCFrameChanged` signal. `setUserHeadCFrame(value)` also raises a property changed event separately. both guard with a dirty check.

`recenterUserHeadCFrame()` just sets `recenterUserHeadCFrameRequested = true`. the actual recentering is handled externally by whatever calls `checkAndClearRecenterUserHeadCFrameRequest()` — it's a request/acknowledge pattern rather than an immediate action.

`VREnabled` / `IsVREnabled` — both expose the same `vrEnabled` bool. `IsVREnabled` is deprecated in favor of `VREnabled`.

---

## TextBox Focus Tracking

`getFocusedTextBox()` returns the currently focused TextBox instance, or null if none is focused. it's gated behind `DFFlag::GetFocusedTextBoxEnabled` — if the flag is off, it logs an error and returns null:

```cpp
shared_ptr<Instance> UserInputService::getFocusedTextBox() {
    if (!DFFlag::GetFocusedTextBoxEnabled) {
        StandardOut::singleton()->printf(MESSAGE_ERROR, "UserInputService:GetFocusedTextBox() is not yet enabled");
        return shared_ptr<Instance>();
    }
    return currentTextBox;
}
```

focus tracking is wired up in `onServiceProvider` — `textBoxGainFocus` and `textBoxReleaseFocus` events connect to `textboxFocused`/`textboxFocusReleased`, which set/clear `currentTextBox`. this connection is only made if the flag is enabled.

`TextBoxFocused` and `TextBoxFocusReleased` events are public and always fire regardless of the flag.

`textboxDidFinishEditing` queues a `(text, shouldReturn)` pair into `textboxFinishedVector` (with mutex), and `processTextboxInternal` drains it and fires `textBoxFinishedEditing`. if `shouldReturn` is true, a dummy `InputObject` for the Return key is passed as the third argument.

---

## Hidden Debug Cheat Codes

`showStatsBasedOnInputString(text)` is called when a TextBox submits text. it checks for hardcoded strings and toggles internal debug overlays:

| Text | Effect |
|---|---|
| `Genstats` | toggles general stats panel |
| `Renstats` | toggles render stats |
| `Netstats` | toggles network stats |
| `Phystats` | toggles physics stats |
| `Sumstats` | toggles summary stats |
| `Cusstats` | toggles custom stats |
| `console` | toggles dev console (case-insensitive) |

the whole function short-circuits and returns false if `DFFlag::EnableShowStatsLua` is on — that flag was probably added to eventually deprecate this mechanism. "console" is the only case-insensitive one (uses `boost::iequals`).

---

## Touch Debug Rendering

there's a static bool `DrawTouchEvents` (default off) and two color statics `DrawTouchColor` and `DrawMoveColor`. when enabled, every touch event clones a 4×4 pixel `Frame` and places it in `CoreGui/RobloxGui` at the touch position. move events cycle through three colors (green → cyan → yellow) to make them visually distinct from begin (red) and end (blue) events. this is all in the `processInputVector` code path and would only be useful for internal engine debugging.

---

## Quick Reference

### Properties (read-only from Lua)

| Property | Type | Notes |
|---|---|---|
| `TouchEnabled` | bool | true on mobile or Studio emulation |
| `KeyboardEnabled` | bool | true on desktop |
| `MouseEnabled` | bool | true on desktop |
| `GamepadEnabled` | bool | true if any gamepad connected |
| `AccelerometerEnabled` | bool | set by hardware layer |
| `GyroscopeEnabled` | bool | set by hardware layer |
| `MouseIconEnabled` | bool | readable and writable |
| `MouseBehavior` | enum | Default / LockCenter / LockCurrentPosition |
| `VREnabled` | bool | set by client |
| `UserHeadCFrame` | CFrame | deprecated, use GetUserCFrame |
| `ModalEnabled` | bool | readable and writable |

### Events

| Event | Fires When |
|---|---|
| `InputBegan` | any input starts |
| `InputChanged` | any input changes |
| `InputEnded` | any input ends |
| `TouchStarted/Moved/Ended` | touch-specific |
| `TouchTap/LongPress/Swipe/Pinch/Rotate/Pan` | gesture recognized |
| `GamepadConnected/Disconnected` | gamepad plugged in or out |
| `DeviceAccelerationChanged` | accelerometer data |
| `DeviceGravityChanged` | gravity vector data |
| `DeviceRotationChanged` | gyro data |
| `LastInputTypeChanged` | last input type changed |
| `TextBoxFocused/FocusReleased` | text box focus changed |
| `WindowFocused/FocusReleased` | app window focus |
| `JumpRequest` | jump requested |
| `UserCFrameChanged` | VR CFrame updated |

### Functions

| Function | Notes |
|---|---|
| `IsKeyDown(keyCode)` | checks current key state |
| `GetKeysPressed()` | all currently held keys |
| `GetConnectedGamepads()` | array of connected gamepad types |
| `GetGamepadConnected(type)` | true if specific gamepad connected |
| `GetGamepadState(type)` | array of InputObjects for gamepad |
| `GamepadSupports(type, keyCode)` | true if gamepad has that button |
| `GetSupportedGamepadKeyCodes(type)` | full list of supported codes |
| `GetNavigationGamepads()` | gamepads in navigation mode |
| `GetUserCFrame(type)` | VR CFrame for Head/LeftHand/RightHand |
| `RecenterUserHeadCFrame()` | requests a VR recenter |
| `GetFocusedTextBox()` | currently focused TextBox (flag-gated) |
| `GetDeviceAcceleration()` | current accelerometer InputObject |
| `GetDeviceGravity()` | current gravity InputObject |
| `GetDeviceRotation()` | (rotationInputObject, CFrame) tuple |
| `GetLastInputType()` | last active input type |
| `GetPlatform()` | current platform enum (RobloxScript only) |

---

