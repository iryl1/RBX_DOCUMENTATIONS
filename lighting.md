# Roblox Lighting System Docs

Lighting system — how it works, what all the types and properties actually mean, and how to replicate it by writing to offsets yourself
---


## What this covers

- how time is stored and converted
- sun + moon positioning — only the matrix-rotated "true" positions are stored and accessible
- how the game picks between sun and moon as the active light source
- color interpolation throughout the day
- **how to replicate this yourself by writing to roblox property offsets**

> **note on missing stuff:** moon phases and star field rotation exist in older source but are no longer accessible as far as i can tell. star field definitely isn't. moon phases might be, but i haven't confirmed it so i've left both out for now. if you figure it out lmk.

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

the game only stores and exposes the **"true" sun position** — which is the basic circular position after being rotated by a matrix to account for earth's axial tilt, time of year, and geographic latitude. you don't get access to the intermediate unrotated position.

```cpp
float dayOfYearOffset = (time - (time * floor(time / solarYear))) / DAY;
float latRad = toRadians(geoLatitude);
float sunOffset = -earthTilt * cos(π * (dayOfYearOffset - halfSolarYear) / halfSolarYear) - latRad;
Matrix3 rotMat = Matrix3::fromAxisAngle(Vector3::unitZ().cross(sunPosition), sunOffset);
trueSunPosition = rotMat * sunPosition;
```

the rotation accounts for earth's 23.5° axial tilt, what time of year it is, and your geographic latitude. `trueSunPosition` is what gets stored — that's what you read and write.

---

## Moon Position

```cpp
moonPosition.x = sin(sourceAngle + π);
moonPosition.y = -cos(sourceAngle + π);
moonPosition.z = 0;
```

the moon is literally just the sun flipped 180°. makes sense when you think about it.

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


## Sky color throughout the day

the sky isn't a single flat color — it's a **gradient** with a top and bottom value. you'll see these referred to as either `SkyColor` / `SkyColor2` or `SkyGradientTop` / `SkyGradientBottom` depending on the context. both pairs mean the same thing.

there's also **LightColor** (the color of the active light source, i.e. sun or moon) and **LightDirection** (a Vector3 pointing toward the active light source) which are separate from sky color entirely.

the sky colors interpolate across the day using the same keyframes:

| Time     | Color (RGB)           | vibe            |
|----------|-----------------------|-----------------|
| Midnight | `(0.20, 0.20, 0.20)` | dark grey       |
| Sunrise  | `(0.60, 0.60, 0.00)` | orange          |
| Noon     | `(0.75, 0.75, 0.75)` | bright white    |
| Sunset   | `(0.10, 0.05, 0.05)` | deep red-orange |

top and bottom of the gradient follow different keyframes — the bottom tends to be warmer/lighter near the horizon while the top is darker. you'll want to dump the actual keyframe values yourself since these vary.

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

    uintptr_t TrueSunDir      = 0x???;  // Vector3 — matrix-rotated sun position
    uintptr_t TrueMoonDir     = 0x???;  // Vector3 — matrix-rotated moon position
    uintptr_t SkyColorTop     = 0x???;  // Color3 — sky gradient top (also called SkyColor)
    uintptr_t SkyColorBottom  = 0x???;  // Color3 — sky gradient bottom (also called SkyColor2)
    uintptr_t LightColor      = 0x???;  // Color3 — color of the active light source
    uintptr_t LightDirection  = 0x???;  // Vector3 — direction toward active light source
    uintptr_t Brightness      = 0x???;  // float
    uintptr_t ActiveSource    = 0x???;  // int (0 = sun, 1 = moon)
}

HANDLE hProcess = nullptr;
uintptr_t moduleBase = 0;
```





### deriving all lighting properties from a time value

you pass in a time (0-24 hours) and this computes everything the system would normally calculate internally:

```cpp
// applies the axial tilt + latitude rotation to a basic circular position
// returns result as a float[3] — x, y, z
void ApplyTiltRotation(float* out, float bx, float by, float bz, float tiltRad) {
    out[0] = bx;
    out[1] = by * cosf(tiltRad) - bz * sinf(tiltRad);
    out[2] = by * sinf(tiltRad) + bz * cosf(tiltRad);
}

void CalcTrueSunDir(float* out, float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    double angle = 2.0 * M_PI * (clockTime / 24.0);
    float bx = (float)sin(angle);
    float by = (float)-cos(angle);
    float bz = 0.0f;

    float halfYear  = 182.6282f;
    float tilt      = -0.4101f * cosf(M_PI * (dayOfYear - halfYear) / halfYear);
    float latRad    = geoLatDeg * (M_PI / 180.0f);
    ApplyTiltRotation(out, bx, by, bz, tilt - latRad);
}

void CalcTrueMoonDir(float* out, float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    double angle = 2.0 * M_PI * (clockTime / 24.0) + M_PI;
    float bx = (float)sin(angle);
    float by = (float)-cos(angle);
    float bz = 0.0f;

    float halfYear  = 182.6282f;
    float tilt      = -0.4101f * cosf(M_PI * (dayOfYear - halfYear) / halfYear);
    float latRad    = geoLatDeg * (M_PI / 180.0f);
    ApplyTiltRotation(out, bx, by, bz, tilt - latRad);
}

bool CalcIsDaytime(float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    float sun[3];
    CalcTrueSunDir(sun, clockTime, geoLatDeg, dayOfYear);
    return sun[1] > -0.3f; // y component
}

// lerp between two float[3] colors by t (0-1), result written to out
void LerpColor(float* out, float* a, float* b, float t) {
    out[0] = a[0] + (b[0] - a[0]) * t;
    out[1] = a[1] + (b[1] - a[1]) * t;
    out[2] = a[2] + (b[2] - a[2]) * t;
}

// approximate sky color at a given time — midnight->sunrise->noon->sunset->midnight
void CalcSkyColor(float* out, float clockTime) {
    float midnight[3] = { 0.20f, 0.20f, 0.20f };
    float sunrise[3]  = { 0.60f, 0.60f, 0.00f };
    float noon[3]     = { 0.75f, 0.75f, 0.75f };
    float sunset[3]   = { 0.10f, 0.05f, 0.05f };

    if      (clockTime < 6.0f)  LerpColor(out, midnight, sunrise, clockTime / 6.0f);
    else if (clockTime < 12.0f) LerpColor(out, sunrise,  noon,    (clockTime - 6.0f)  / 6.0f);
    else if (clockTime < 18.0f) LerpColor(out, noon,     sunset,  (clockTime - 12.0f) / 6.0f);
    else                        LerpColor(out, sunset,   midnight, (clockTime - 18.0f) / 6.0f);
}

// brightness peaks at noon (1.0), dips at night (0.1)
float CalcBrightness(float clockTime) {
    double angle = 2.0 * M_PI * (clockTime / 24.0);
    float sunY = (float)-cos(angle);
    float t = (sunY + 1.0f) / 2.0f;
    return 0.1f + t * 0.9f;
}


```

### writing everything to the game

now you just call all of those and write each result to its own offset:

```cpp
void SetLightingFromTime(uintptr_t base, float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    float sunDir[3], moonDir[3], skyColor[3];
    CalcTrueSunDir(sunDir,   clockTime, geoLatDeg, dayOfYear);
    CalcTrueMoonDir(moonDir, clockTime, geoLatDeg, dayOfYear);
    CalcSkyColor(skyColor, clockTime);

    float brightness = CalcBrightness(clockTime);
    bool  daytime    = CalcIsDaytime(clockTime, geoLatDeg, dayOfYear);
    int   source     = daytime ? 0 : 1;

    // light direction is just whichever source is active
    float* lightDir = daytime ? sunDir : moonDir;

    // sky gradient — bottom is slightly warmer than top
    float skyBottom[3] = { skyColor[0] * 1.15f, skyColor[1] * 1.1f, skyColor[2] * 1.0f };

    // light color — warm yellow day, cool blue night
    float lightColor[3] = daytime
        ? (float[3]){ 1.0f, 0.95f, 0.8f }
        : (float[3]){ 0.6f, 0.6f,  0.9f };

    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueSunDir),     sunDir,     sizeof(float) * 3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueMoonDir),    moonDir,    sizeof(float) * 3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyColorTop),    skyColor,   sizeof(float) * 3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyColorBottom), skyBottom,  sizeof(float) * 3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LightColor),     lightColor, sizeof(float) * 3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LightDirection), lightDir,   sizeof(float) * 3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Brightness),     &brightness,sizeof(float),     nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ActiveSource),   &source,    sizeof(int),       nullptr);
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
const double HOUR     = 3600;
const double DAY      = 86400;
const double SUNRISE  = 6  * HOUR;
const double SUNSET   = 18 * HOUR;
const double NOON     = 12 * HOUR;
const double MIDNIGHT = 0;
```

---

*more stuff coming as i figure it out. PRs welcome if you spot something wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
