# Roblox Lighting System Docs

so i got tired of digging through forums and discord servers trying to figure out how roblox's lighting actually works under the hood, so i just started writing it down myself. this is what i've figured out so far — if something's wrong or missing lmk and i'll fix it.

if this helps you out at all, drop a ⭐ or come hang in the discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)

---

## What this covers

- how time is stored and converted
- sun + moon positioning (both the simple version and the "physically correct" one)
- moon phases
- star field rotation
- how the game picks between sun and moon as the active light source
- color interpolation throughout the day
- **how to replicate this yourself by writing to roblox property offsets**

---

## Time System

time is stored internally as **seconds since midnight**. pretty straightforward once you know that, but it tripped me up at first.

### getting minutes from the internal time

```cpp
double getMinutesAfterMidnight() {
    return timeOfDay.total_milliseconds() / 60000.0;
}
```

just divides milliseconds by 60,000 to get minutes. nothing fancy.

### setting the time

```cpp
void setMinutesAfterMidnight(double value) {
    setTime(boost::posix_time::seconds((int)(60 * value)));
}
```

converts minutes back to seconds for storage. so if you pass in `870` you get 2:30 PM.

### wrapping to 24 hours

```cpp
void setTime(const boost::posix_time::time_duration& value) {
    int seconds = value.total_seconds();
    seconds %= 60 * 60 * 24; // keeps it within 0-86399
    timeOfDay = boost::posix_time::seconds(seconds);
}
```

86400 = 24 × 60 × 60. the modulo just makes sure time wraps around instead of going past midnight into weird territory.

---

## Sun Position

there are two modes — a simple one and a more realistic one.

### basic mode

```cpp
float sourceAngle = 2 * π * time / DAY;
sunPosition.x = sin(sourceAngle);
sunPosition.y = -cos(sourceAngle);
sunPosition.z = 0;
```

this gives you a clean circular arc across the sky. at midnight the sun is at `(0, -1, 0)` (underground basically), at noon it's at `(0, 1, 0)` directly overhead. simple but it works.

### physically correct mode

```cpp
float dayOfYearOffset = (time - (time * floor(time / solarYear))) / DAY;
float latRad = toRadians(geoLatitude);
float sunOffset = -earthTilt * cos(π * (dayOfYearOffset - halfSolarYear) / halfSolarYear) - latRad;
Matrix3 rotMat = Matrix3::fromAxisAngle(Vector3::unitZ().cross(sunPosition), sunOffset);
trueSunPosition = rotMat * sunPosition;
```

this one actually accounts for earth's 23.5° axial tilt, what time of year it is, and your geographic latitude. if you're just making a normal game you probably don't need this, but it's there.

---

## Moon Position

```cpp
moonPosition.x = sin(sourceAngle + π);
moonPosition.y = -cos(sourceAngle + π);
moonPosition.z = 0;
```

the moon is literally just the sun flipped 180°. makes sense when you think about it.

### moon phases

```cpp
moonPhase = floor(time / moonPhaseInterval) + initialMoonPhase;
// moonPhaseInterval = 29.53 days = 2,549,520 seconds
```

one full lunar cycle every 29.53 days. the `+ 0.75` offset (`initialMoonPhase`) just controls what phase it starts on.

---

## Which light source is active?

```cpp
if (!physicallyCorrect) {
    if ((sourceAngle < π/2) || (sourceAngle > 3π/2)) {
        source = MOON;
    } else {
        source = SUN;
    }
} else {
    if (trueSunPosition.y > -0.3f) {
        source = SUN;
    } else {
        source = MOON;
    }
}
```

in basic mode it just checks the angle. in physical mode it uses `y > -0.3` as the cutoff — that small buffer means the sun still lights the scene a little even when it's just below the horizon, which is what gives you that twilight effect.

---

## Star Field

```cpp
double starRot = initialStarRot - (2π * (time - (time * floor(time / SIDEREAL_DAY))) / SIDEREAL_DAY);
starVec.x = cos(starRot);
starVec.y = 0;
starVec.z = sin(starRot);
```

stars use a **sidereal day** (86,164 seconds = 23h 56m 4s) instead of a solar day. that's why they complete a full rotation slightly faster than the sun — same as in real life.

---

## Sky color throughout the day

the game uses spline interpolation to smoothly blend between these:

| Time     | Color (RGB)           | vibe            |
|----------|-----------------------|-----------------|
| Midnight | `(0.20, 0.20, 0.20)` | dark grey       |
| Sunrise  | `(0.60, 0.60, 0.00)` | orange          |
| Noon     | `(0.75, 0.75, 0.75)` | bright white    |
| Sunset   | `(0.10, 0.05, 0.05)` | deep red-orange |

---

## Replicating it yourself (writing to offsets)

this is the part most people actually want. there's no single ClockTime param you can just write to — to replicate lighting at a given time you have to **derive every property from the time yourself** using the formulas, then write each one to its own offset separately. sun direction, moon direction, brightness, sky color — all of it, individually.

> **note:** all offsets below are **placeholders** — swap them out for the real ones you've dumped yourself. they shift between updates so don't expect these to work out of the box.

### setup

```cpp
#include <Windows.h>
#include <TlHelp32.h>
#include <cmath>
#include <iostream>

// placeholder offsets — replace with your own dumped values
namespace Offsets {
    uintptr_t RenderView      = 0x????????; // base of the lighting/render object

    // sun direction vector — written as 3 separate floats
    uintptr_t SunDirX         = 0x???;
    uintptr_t SunDirY         = 0x???;
    uintptr_t SunDirZ         = 0x???;

    // moon direction vector — same deal
    uintptr_t MoonDirX        = 0x???;
    uintptr_t MoonDirY        = 0x???;
    uintptr_t MoonDirZ        = 0x???;

    // sky/ambient color at current time — RGB as separate floats
    uintptr_t SkyColorR       = 0x???;
    uintptr_t SkyColorG       = 0x???;
    uintptr_t SkyColorB       = 0x???;

    // brightness of the active light source
    uintptr_t Brightness      = 0x???;

    // which source is active (0 = sun, 1 = moon)
    uintptr_t ActiveSource    = 0x???;

    // star field rotation
    uintptr_t StarRotX        = 0x???;
    uintptr_t StarRotZ        = 0x???;
}

HANDLE hProcess = nullptr;
uintptr_t moduleBase = 0;
```

### getting the process handle

```cpp
HANDLE GetRobloxHandle() {
    HWND hwnd = FindWindowA(nullptr, "Roblox");
    if (!hwnd) return nullptr;

    DWORD pid;
    GetWindowThreadProcessId(hwnd, &pid);
    return OpenProcess(PROCESS_VM_READ | PROCESS_VM_WRITE | PROCESS_VM_OPERATION, FALSE, pid);
}

uintptr_t GetModuleBase(HANDLE hProc, const char* modName) {
    HMODULE hMods[1024];
    DWORD cbNeeded;
    if (EnumProcessModules(hProc, hMods, sizeof(hMods), &cbNeeded)) {
        for (unsigned i = 0; i < cbNeeded / sizeof(HMODULE); i++) {
            char name[MAX_PATH];
            GetModuleBaseNameA(hProc, hMods[i], name, sizeof(name));
            if (strcmp(name, modName) == 0)
                return (uintptr_t)hMods[i];
        }
    }
    return 0;
}
```

### read/write helpers

```cpp
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

### deriving all lighting properties from a time value

you pass in a time (0-24 hours) and this computes everything the system would normally calculate internally:

```cpp
struct Vec3  { float x, y, z; };
struct Color { float r, g, b; };

// 0-24 hours -> angle -> sun direction
// same formula as the internal system: x=sin(angle), y=-cos(angle)
Vec3 CalcSunDir(float clockTime) {
    double angle = 2.0 * M_PI * (clockTime / 24.0);
    return { (float)sin(angle), (float)-cos(angle), 0.0f };
}

// moon is always 180 degrees opposite the sun
Vec3 CalcMoonDir(float clockTime) {
    double angle = 2.0 * M_PI * (clockTime / 24.0) + M_PI;
    return { (float)sin(angle), (float)-cos(angle), 0.0f };
}

// true if sun is above the horizon (y > 0)
bool CalcIsDaytime(float clockTime) {
    return CalcSunDir(clockTime).y > 0.0f;
}

// lerp between two colors by t (0-1)
Color LerpColor(Color a, Color b, float t) {
    return {
        a.r + (b.r - a.r) * t,
        a.g + (b.g - a.g) * t,
        a.b + (b.b - a.b) * t
    };
}

// approximate sky color at a given time using the same keyframes the system uses
// midnight(0) -> sunrise(6) -> noon(12) -> sunset(18) -> midnight(24)
Color CalcSkyColor(float clockTime) {
    Color midnight = { 0.20f, 0.20f, 0.20f };
    Color sunrise  = { 0.60f, 0.60f, 0.00f };
    Color noon     = { 0.75f, 0.75f, 0.75f };
    Color sunset   = { 0.10f, 0.05f, 0.05f };

    if (clockTime < 6.0f)       // midnight -> sunrise
        return LerpColor(midnight, sunrise, clockTime / 6.0f);
    else if (clockTime < 12.0f) // sunrise -> noon
        return LerpColor(sunrise, noon, (clockTime - 6.0f) / 6.0f);
    else if (clockTime < 18.0f) // noon -> sunset
        return LerpColor(noon, sunset, (clockTime - 12.0f) / 6.0f);
    else                        // sunset -> midnight
        return LerpColor(sunset, midnight, (clockTime - 18.0f) / 6.0f);
}

// brightness peaks at noon, dips at night
float CalcBrightness(float clockTime) {
    Vec3 sun = CalcSunDir(clockTime);
    // sun.y goes from -1 (midnight) to +1 (noon)
    // remap to a 0.1 - 1.0 brightness range
    float t = (sun.y + 1.0f) / 2.0f;
    return 0.1f + t * 0.9f;
}

// sidereal star rotation — completes one cycle every 86164s (23h 56m 4s)
// slightly faster than the solar day, same as real life
Vec3 CalcStarRot(float clockTime, float initialStarRot = 0.0f) {
    double timeSeconds = clockTime * 3600.0;
    double siderealDay = 86164.0;
    double rot = initialStarRot - (2.0 * M_PI * fmod(timeSeconds, siderealDay) / siderealDay);
    return { (float)cos(rot), 0.0f, (float)sin(rot) };
}
```

### writing everything to the game

now you just call all of those and write each result to its own offset:

```cpp
void SetLightingFromTime(uintptr_t base, float clockTime) {
    Vec3  sunDir    = CalcSunDir(clockTime);
    Vec3  moonDir   = CalcMoonDir(clockTime);
    Color skyColor  = CalcSkyColor(clockTime);
    float brightness= CalcBrightness(clockTime);
    bool  daytime   = CalcIsDaytime(clockTime);
    Vec3  starRot   = CalcStarRot(clockTime);

    // sun direction
    Write<float>(base + Offsets::SunDirX, sunDir.x);
    Write<float>(base + Offsets::SunDirY, sunDir.y);
    Write<float>(base + Offsets::SunDirZ, sunDir.z);

    // moon direction
    Write<float>(base + Offsets::MoonDirX, moonDir.x);
    Write<float>(base + Offsets::MoonDirY, moonDir.y);
    Write<float>(base + Offsets::MoonDirZ, moonDir.z);

    // sky color
    Write<float>(base + Offsets::SkyColorR, skyColor.r);
    Write<float>(base + Offsets::SkyColorG, skyColor.g);
    Write<float>(base + Offsets::SkyColorB, skyColor.b);

    // brightness + active source
    Write<float>(base + Offsets::Brightness,   brightness);
    Write<int>  (base + Offsets::ActiveSource, daytime ? 0 : 1);

    // star rotation
    Write<float>(base + Offsets::StarRotX, starRot.x);
    Write<float>(base + Offsets::StarRotZ, starRot.z);
}
```

### putting it all together

```cpp
int main() {
    hProcess = GetRobloxHandle();
    if (!hProcess) {
        std::cout << "couldn't find roblox, is it running?\n";
        return 1;
    }

    moduleBase = GetModuleBase(hProcess, "RobloxPlayerBeta.exe");
    uintptr_t lightingBase = moduleBase + Offsets::RenderView;

    // set to noon
    SetLightingFromTime(lightingBase, 12.0f);
    std::cout << "set lighting to noon\n";

    // set to sunset
    SetLightingFromTime(lightingBase, 18.0f);
    std::cout << "set lighting to sunset\n";

    // set to 3am
    SetLightingFromTime(lightingBase, 3.0f);
    std::cout << "set lighting to 3am\n";

    CloseHandle(hProcess);
    return 0;
}
```

### quick reference — what the formulas produce at key times

| Time (hrs) | Sun y   | Day/Night | Brightness | Sky color       |
|------------|---------|-----------|------------|-----------------|
| `0` / `24` | `-1.0`  | night     | `0.10`     | dark grey       |
| `6.0`      | `0.0`   | threshold | `0.55`     | orange          |
| `12.0`     | `+1.0`  | day       | `1.00`     | bright white    |
| `18.0`     | `0.0`   | threshold | `0.55`     | deep red-orange |

the key thing to understand is that **every visual property is derived from the same angle** — `2π × (clockTime / 24)`. once you have that angle, everything else follows from it.

---

## Constants worth knowing

```cpp
const double HOUR         = 3600;
const double DAY          = 86400;
const double SIDEREAL_DAY = 86164;  // 23h 56m 4s
const double SUNRISE      = 6  * HOUR;
const double SUNSET       = 18 * HOUR;
const double NOON         = 12 * HOUR;
const double MIDNIGHT     = 0;
```

---

*more stuff coming as i figure it out. PRs welcome if you spot something wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
