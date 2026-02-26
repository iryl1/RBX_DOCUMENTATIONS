# Roblox Camera System Docs

continuing the internal docs series. this one covers the camera system — how it works, what all the types and properties actually mean, and how to replicate it by writing to offsets yourself. same deal as before, offsets are placeholders.

if this helps, ⭐ star the repo or join the discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)

---

## What this covers

- camera types and what each one actually does
- how position, focus, and coordinate frames work
- fov, roll, zoom, pan, tilt
- frustum and ray casting
- interpolation / lerp system
- how to replicate by writing each property to its own offset

---

## Camera Types

there are 7 camera types stored as an enum internally. important note from the source: **these are part of the XML serialization, so they never change values** — only new ones get appended.

| Value | Name       | Internal Name    | What it does |
|-------|------------|------------------|--------------|
| `0`   | Fixed      | `FIXED_CAMERA`   | camera doesn't move at all |
| `1`   | Attach     | `ATTACH_CAMERA`  | locks position + rotation relative to the target's coordinate frame |
| `2`   | Watch      | `WATCH_CAMERA`   | rotates to keep the target in view, doesn't translate |
| `3`   | Track      | `TRACK_CAMERA`   | translates to keep target in view, doesn't rotate |
| `4`   | Follow     | `FOLLOW_CAMERA`  | rotates AND translates to keep target in view |
| `5`   | Custom     | `CUSTOM_CAMERA`  | script-controlled |
| `6`   | Scriptable | `LOCKED_CAMERA`  | fully locked, exposed to Lua as "Scriptable" |

`CUSTOM_CAMERA` and `LOCKED_CAMERA` are both script-controlled — the difference is that Custom still partially interacts with the character system while Locked is fully hands-off.

---

## Camera Modes and Pan Modes

### CameraMode

| Value | Name              | What it does |
|-------|-------------------|--------------|
| `0`   | Classic           | normal third person |
| `1`   | LockFirstPerson   | forces first person, locks mouse |

### CameraPanMode

| Value | Name      | What it does |
|-------|-----------|--------------|
| `0`   | Classic   | pan with mouse drag |
| `1`   | EdgeBump  | pan when cursor hits screen edge |

---

## Hardcoded Distance Constants

these are baked in as static values — they don't come from a config file:

```cpp
distanceDefault()      = 36.0f   // studs, starting zoom distance
distanceMin()          = 0.5f    // closest you can zoom in
distanceMax()          = 1000.0f // furthest zoom allowed
distanceMaxCharacter() = 400.0f  // max zoom when a character is attached
distanceMinOcclude()   = 4.0f    // below this, occlusion checks stop
interpolationSpeed()   = 5.0f    // studs per second for constant-speed lerp
```

---

## Field of View

fov is stored internally in **radians**, but exposed to lua in **degrees**. the default is 70°.

```cpp
// internal default
const float defaultFieldOfView = G3D::toRadians(70.0f); // ~1.2217 radians

// image plane depth is derived from fov at construction time:
imagePlaneDepth = 1.0f / (2.0f * tanf(defaultFieldOfView / 2.0f));
```

the image plane depth is what the projection math uses to figure out where things land on screen. it's recalculated whenever fov changes.

---

## Coordinate Frames — what's what

the camera has several CoordinateFrames stored internally. this tripped me up so here's the breakdown:

| Field             | What it is |
|-------------------|------------|
| `cameraCoord`     | where the camera actually is right now |
| `cameraFocus`     | what the camera is looking at right now |
| `cameraCoordGoal` | where we're trying to get to (lerp target) |
| `cameraFocusGoal` | what we're trying to look at (lerp target) |
| `cameraCoordPrev` | where we were before (used during interpolation) |
| `cameraFocusPrev` | what we were looking at before (interpolation) |

when you set `CFrame` through Lua, you're writing to `cameraCoord`. `Focus` maps to `cameraFocus`.

the **rendering** coordinate frame is different — it applies roll and VR head offset on top of `cameraCoord`:

```cpp
CoordinateFrame getRenderingCoordinateFrame() {
    CoordinateFrame result = cameraCoord;

    // apply roll if nonzero
    if (roll != 0)
        result.rotation *= Matrix3::fromAxisAngle(Vector3::unitZ(), -roll);

    // apply VR head CFrame if headLocked is true
    if (headLocked)
        result = result * uis->getUserHeadCFrame();

    return result;
}
```

so if you're reading camera position for rendering purposes, you need the rendering frame, not raw `cameraCoord`.

---

## Zoom Formula

zoom isn't linear — it uses a multiplier based on current distance:

```cpp
float getNewZoomDistance(float currentDistance, float in) {
    const float ZOOM_FACTOR = 0.25f;

    if (in > 0.0f)  // zooming in
        return max(currentDistance / (1.0f + ZOOM_FACTOR * in), distanceMin());
    else if (in < 0.0f)  // zooming out
        return min(currentDistance * (1.0f - ZOOM_FACTOR * in), distanceMax());
    else
        return currentDistance;
}
```

zooming in divides distance, zooming out multiplies it. the further out you are, the bigger each scroll step feels — which is intentional.

---

## Frustum

the frustum is a truncated pyramid representing everything the camera can see. it's recalculated from fov, viewport, and the rendering coordinate frame:

```cpp
// horizontal fov derived from vertical fov + aspect ratio
float fovx = 2 * atan(tan(fieldOfView * 0.5f) * viewport.x / viewport.y);

// near/far plane z values (both negative, camera looks down -Z)
nearPlaneZ() // calculated based on scene, not hardcoded
farPlaneZ()  = -5000.0f  // hardcoded, 5000 studs
```

---

## Projection Math

the projection matrix maps 3D world space to 2D screen space:

```cpp
float h = 1 / tan(fieldOfView / 2);   // vertical scale
float w = h / aspect;                  // horizontal scale (adjusted for aspect ratio)

// maps Z to [0..1] range
float q  = -zfar / (zfar - znear);
float qn = znear * q;

Matrix4 projection(
    w, 0, 0, 0,
    0, h, 0, 0,
    0, 0, q, qn,
    0, 0, -1, 0
);
```

to project a world point to screen:

```cpp
Vector4 projected = projection * view * Vector4(worldPoint, 1.0f);
Vector3 ndc = projected.xyz() / projected.w;  // normalize

float screenX = (width  / 2.0f) + ((width  / 2.0f) * ndc.x);
float screenY = (height / 2.0f) - ((height / 2.0f) * ndc.y);  // Y is flipped
```

if `projected.w <= 0`, the point is behind the camera — the engine returns `Vector3::inf()` in that case.

---

## World Ray (screen point → world direction)

this is how `ScreenPointToRay` and `ViewportPointToRay` work internally. the difference between the two is that `ScreenPointToRay` applies the GUI inset offset first:

```cpp
RbxRay worldRay(float x, float y, float depth) {
    float cx = viewport.x / 2.0f;
    float cy = viewport.y / 2.0f;

    // convert pixel to NDC
    Vector3 point = Vector3((x / cx) - 1.0f, 1.0f - (y / cy), imagePlaneDepth);

    // unproject through inverse projection matrix
    Vector4 unprojected = projection.inverse() * Vector4(point, 1.0f);
    Vector3 direction = (unprojected.xyz() / unprojected.w) - origin;
    direction = direction.direction(); // normalize

    // offset origin to near clip plane
    float theta = acos(direction.dot(cameraFrame.lookVector()));
    float depthToNear = imagePlaneDepth / sin((π / 2) - theta);

    return Ray(origin + direction * depthToNear + direction * depth, direction);
}
```

---

## Interpolation System

the camera has three interpolation modes internally:

| Mode | What it does |
|------|--------------|
| `CAM_INTERPOLATION_CONSTANT_TIME` | lerps over a fixed duration in seconds |
| `CAM_INTERPOLATION_CONSTANT_SPEED` | lerps at a fixed speed (5 studs/sec), duration depends on distance |
| `CAM_INTERPOLATION_NONE` | no interpolation, snaps immediately |

when interpolation finishes it fires `interpolationFinishedSignal` (exposed to Lua as `InterpolationFinished`).

---

## Replicating it yourself (writing to offsets)

same approach as the lighting docs — there's no single property you can write to set the camera state. you have to derive each value and write them individually.

> **note:** all offsets are placeholders. dump your own.

### setup

```cpp
#include <Windows.h>
#include <TlHelp32.h>
#include <cmath>

namespace Offsets {
    uintptr_t CameraBase        = 0x????????; // base of the Camera object

    // CFrame / CoordinateFrame — position + rotation matrix (stored as 4x3)
    uintptr_t CameraCoord_Pos   = 0x???;  // Vector3 translation (x, y, z floats)
    uintptr_t CameraCoord_Rot   = 0x???;  // Matrix3 rotation (9 floats, row major)

    // Focus CoordinateFrame
    uintptr_t CameraFocus_Pos   = 0x???;
    uintptr_t CameraFocus_Rot   = 0x???;

    // Scalar properties
    uintptr_t FieldOfView       = 0x???;  // float, stored in RADIANS internally
    uintptr_t Roll              = 0x???;  // float, radians
    uintptr_t CameraType        = 0x???;  // int, see enum above
    uintptr_t HeadLocked        = 0x???;  // bool

    // Viewport
    uintptr_t ViewportX         = 0x???;  // int16
    uintptr_t ViewportY         = 0x???;  // int16

    // Pan/tilt speeds
    uintptr_t PanSpeed          = 0x???;  // float
    uintptr_t TiltSpeed         = 0x???;  // float

    // Interpolation
    uintptr_t InterpDuration    = 0x???;  // float, seconds
    uintptr_t InterpTime        = 0x???;  // float, elapsed interpolation time
    uintptr_t InterpMode        = 0x???;  // int, 0=constant time, 1=constant speed, 2=none
}

HANDLE hProcess = nullptr;

template<typename T>
T Read(uintptr_t address) {
    T val{};
    ReadProcessMemory(hProcess, (LPCVOID)address, &val, sizeof(T), nullptr);
    return val;
}

template<typename T>
void Write(uintptr_t address, T value) {
    WriteProcessMemory(hProcess, (LPVOID)address, &value, sizeof(T), nullptr);
}
```

### structs

```cpp
struct Vec3 { float x, y, z; };
struct Mat3 { float m[3][3]; }; // row major

struct CoordFrame {
    Mat3 rotation;
    Vec3 translation;
};
```

### reading the current camera state

```cpp
CoordFrame ReadCFrame(uintptr_t base) {
    CoordFrame cf{};
    ReadProcessMemory(hProcess, (LPCVOID)(base + Offsets::CameraCoord_Rot), &cf.rotation, sizeof(Mat3), nullptr);
    ReadProcessMemory(hProcess, (LPCVOID)(base + Offsets::CameraCoord_Pos), &cf.translation, sizeof(Vec3), nullptr);
    return cf;
}

float ReadFOV(uintptr_t base) {
    // internally radians — convert to degrees if you want
    float fovRad = Read<float>(base + Offsets::FieldOfView);
    return fovRad * (180.0f / 3.14159265f);
}
```

### setting camera position and focus

you can't write a CoordinateFrame as one blob — write position and rotation separately:

```cpp
void WriteCFrame(uintptr_t base, Vec3 pos, Mat3 rot) {
    // position is a Vector3 — write it as one block, not three separate floats
    Write<Vec3>(base + Offsets::CameraCoord_Pos, pos);

    // rotation matrix — also one write
    Write<Mat3>(base + Offsets::CameraCoord_Rot, rot);
}

void WriteFocus(uintptr_t base, Vec3 pos, Mat3 rot) {
    Write<Vec3>(base + Offsets::CameraFocus_Pos, pos);
    Write<Mat3>(base + Offsets::CameraFocus_Rot, rot);
}
```

### setting fov

remember it's stored in radians internally, even though lua uses degrees:

```cpp
void SetFOV(uintptr_t base, float degrees) {
    float radians = degrees * (3.14159265f / 180.0f);
    Write<float>(base + Offsets::FieldOfView, radians);

    // imagePlaneDepth needs to stay in sync with fov
    // formula: 1 / (2 * tan(fov / 2))
    float imagePlaneDepth = 1.0f / (2.0f * tanf(radians / 2.0f));
    // write imagePlaneDepth to its own offset too if you have it
}
```

### setting camera type

```cpp
void SetCameraType(uintptr_t base, int type) {
    // 0=Fixed, 1=Attach, 2=Watch, 3=Track, 4=Follow, 5=Custom, 6=Scriptable
    Write<int>(base + Offsets::CameraType, type);
}
```

### building a look-at rotation matrix

if you want to point the camera at something, you need to build the rotation matrix yourself:

```cpp
Mat3 LookAt(Vec3 from, Vec3 to, Vec3 up = {0, 1, 0}) {
    // forward = normalize(to - from), but roblox camera looks down -Z
    Vec3 fwd = {to.x - from.x, to.y - from.y, to.z - from.z};
    float len = sqrtf(fwd.x*fwd.x + fwd.y*fwd.y + fwd.z*fwd.z);
    fwd = {-fwd.x/len, -fwd.y/len, -fwd.z/len}; // negate because -Z forward

    // right = normalize(cross(up, fwd))
    Vec3 right = {
        up.y*fwd.z - up.z*fwd.y,
        up.z*fwd.x - up.x*fwd.z,
        up.x*fwd.y - up.y*fwd.x
    };
    float rlen = sqrtf(right.x*right.x + right.y*right.y + right.z*right.z);
    right = {right.x/rlen, right.y/rlen, right.z/rlen};

    // recalculate up = cross(fwd, right)
    Vec3 newUp = {
        fwd.y*right.z - fwd.z*right.y,
        fwd.z*right.x - fwd.x*right.z,
        fwd.x*right.y - fwd.y*right.x
    };

    Mat3 rot{};
    rot.m[0][0] = right.x;  rot.m[0][1] = newUp.x;  rot.m[0][2] = fwd.x;
    rot.m[1][0] = right.y;  rot.m[1][1] = newUp.y;  rot.m[1][2] = fwd.y;
    rot.m[2][0] = right.z;  rot.m[2][1] = newUp.z;  rot.m[2][2] = fwd.z;
    return rot;
}
```

### putting it all together

```cpp
int main() {
    // get handle (same as lighting docs)
    hProcess = GetRobloxHandle();
    uintptr_t camBase = moduleBase + Offsets::CameraBase;

    // teleport camera to a position looking at origin
    Vec3 camPos  = {0.0f, 50.0f, 100.0f};
    Vec3 focusPos = {0.0f, 0.0f, 0.0f};
    Mat3 rot = LookAt(camPos, focusPos);

    WriteCFrame(camBase, camPos, rot);
    WriteFocus(camBase, focusPos, rot);  // focus rotation usually matches

    // set fov to 90 degrees
    SetFOV(camBase, 90.0f);

    // switch to scriptable (locked) mode so the game doesn't fight your writes
    SetCameraType(camBase, 6);

    // disable head lock (otherwise VR offset gets applied on top of your writes)
    Write<bool>(camBase + Offsets::HeadLocked, false);

    CloseHandle(hProcess);
    return 0;
}
```

one thing worth knowing: if you don't set the camera to `LOCKED_CAMERA` (type 6) first, the game's own camera logic will overwrite your values on the next heartbeat tick. always lock it before writing.

---

## Key defaults (from the constructor)

```cpp
cameraType        = FIXED_CAMERA
cameraFocus       = (0, 0, -5)         // slightly in front of origin
cameraCoord       = (0, 20, 20) looking at (0, 0, 0)
fieldOfView       = toRadians(70.0f)
roll              = 0.0f
panSpeed          = 0.0f
tiltSpeed         = 0.0f
cameraPanMode     = CAMERAPANMODE_CLASSIC
headLocked        = true
interpolation     = CAM_INTERPOLATION_NONE
```

---

*more systems coming. PRs welcome if something's off.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
