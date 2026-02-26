# Roblox Lighting System Docs

Lighting system — how it works, what all the types and properties actually mean, and how to replicate it by writing to offsets yourself
---

## What this covers

- how time is stored and converted
- sun + moon positioning — only the matrix-rotated "true" positions are stored and accessible
- how the game picks between sun and moon as the active light source
- color interpolation throughout the day — lightColor, ambient, diffuseAmbient, skyAmbient, skyAmbient2
- brightness
- lightDirection
- activeSource
- starfield rotation and coordinate frame
- moon phase calculation
- **how to replicate this yourself by writing to roblox property offsets**

> **note on missing stuff:** star field rendering and moon phase *visuals* aren't accessible as far as i can tell, but the underlying calculations for both still run internally and are documented here for completeness. if you find a way to read or write them lmk.

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

internally, `setTime` also calls `skyParameters.setTime(getGameTime())` after updating the clock, which triggers all the sky/sun/moon recalculations. so any lighting property that depends on time of day gets recomputed on every time change.

---

## Sun Position

the game only stores and exposes the **"true" sun position** — which is the basic circular position after being rotated by a matrix to account for earth's axial tilt, time of year, and geographic latitude. you don't get access to the intermediate unrotated position.

```cpp
float dayOfYearOffset = (_time - (_time * floor(_time / solarYear))) / DAY;
float latRad = toRadians(geoLatitude);
float sunOffset = -earthTilt * cos(π * (dayOfYearOffset - halfSolarYear) / halfSolarYear) - latRad;
Matrix3 rotMat = Matrix3::fromAxisAngle(Vector3::unitZ().cross(sunPosition), sunOffset);
trueSunPosition = rotMat * sunPosition;
```

the rotation accounts for earth's 23.5° axial tilt, what time of year it is, and your geographic latitude. `trueSunPosition` is what gets stored — that's what you read and write.

```cpp
void getTrueSunDir(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueSunDir), out, sizeof(float) * 3, nullptr);
}

void setTrueSunDir(uintptr_t base, float x, float y, float z) {
    float dir[3] = { x, y, z };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueSunDir), dir, sizeof(float) * 3, nullptr);
}
```

the base (unrotated) position is just:

```cpp
sunPosition.x = sin(sourceAngle);
sunPosition.y = -cos(sourceAngle);
sunPosition.z = 0;
```

where `sourceAngle = 2π × (time / DAY)`. this is the simple circular orbit before any tilt correction is applied.

---

## Moon Position

the moon is **not** just the sun flipped 180° in physical mode — that's only the basic (non-physical) approximation. in physical mode it has its own orbit driven by moon phase.

### basic moon (non-physical mode)

```cpp
moonPosition.x = sin(sourceAngle + π);
moonPosition.y = -cos(sourceAngle + π);
moonPosition.z = 0;
```

literally just the sun position offset by π. simple.

### true moon position (physical mode)

```cpp
float moonPhase = floor(_time / moonPhaseInterval) + initialMoonPhase;
float moonOffset = ((-earthTilt + moonTilt) * sin(moonPhase * 4)) - latRad;
float curMoonPhase = moonPhase * π * 2;

Vector3 trueMoon = Vector3(
    sin(curMoonPhase + sourceAngle),
    -cos(curMoonPhase + sourceAngle),
    0
);
Matrix3 rotMat = Matrix3::fromAxisAngleFast(Vector3::unitZ().cross(trueMoon), moonOffset);
trueMoonPosition = rotMat * trueMoon;
```

the moon has its own phase-driven offset so it doesn't track exactly opposite the sun. the tilt applied is `(-earthTilt + moonTilt)` — the moon's orbital plane is tilted ~5° off the ecliptic relative to earth's axial tilt.

```cpp
void getTrueMoonDir(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueMoonDir), out, sizeof(float) * 3, nullptr);
}

void setTrueMoonDir(uintptr_t base, float x, float y, float z) {
    float dir[3] = { x, y, z };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueMoonDir), dir, sizeof(float) * 3, nullptr);
}
```

### moon phase

```cpp
static const double moonPhaseInterval = DAY * 29.53; // one lunar cycle
static const double initialMoonPhase  = 0.75;        // phase at Jan 1 1970 midnight

moonPhase = floor(_time / moonPhaseInterval) + initialMoonPhase;
```

the phase is just how many full lunar cycles have passed since epoch, offset by the initial phase value. it increments discretely (floor) rather than continuously. `curMoonPhase = moonPhase * 2π` is what actually feeds into the moon's angular position.

> **note:** moon phase visual rendering doesn't appear to be accessible from outside. the calculation runs but i haven't confirmed a way to read or write the phase visually.

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

`lightDirection` gets set to whichever source is active: `trueSunPosition` during the day, `trueMoonPosition` at night.

---

## Starfield

the starfield has its own coordinate frame that rotates independently from the sun/moon cycle. it's based on the **sidereal day** (23h 56m) rather than the solar day (24h), which is why the stars drift slightly each day relative to the sun.

```cpp
double starRot = initialStarRot - (2 * π * (_time - (_time * floor(_time / SIDEREAL_DAY))) / SIDEREAL_DAY);

starVec.x = cos(starRot);
starVec.y = 0;
starVec.z = sin(starRot);

starFrame.lookAt(starVec, Vector3::unitY());
trueStarFrame.lookAt(starVec, Vector3::unitY());

// apply latitude tilt to true star frame
float aX, aY, aZ;
trueStarFrame.rotation.toEulerAnglesXYZ(aX, aY, aZ);
aX -= geoLatitude;
trueStarFrame.rotation = Matrix3::fromEulerAnglesXYZ(aX, aY, aZ);
```

`starFrame` is the raw rotation. `trueStarFrame` is the same thing with the geographic latitude applied so the north star sits in the right spot relative to your lat/lon.

> **note:** the star field render itself isn't accessible externally as far as i can tell. this is just documenting what the system computes internally.

constants:

```cpp
static const double initialStarRot = 1; // approx star offset at Jan 1 1970 midnight
```

---

## Color System

this is the part that most people get wrong — there isn't just one sky color. there are **five separate color splines** interpolated throughout the day, each covering a different aspect of lighting. all five use `linearSpline` against a set of time keyframes.

the keyframes reference these constants:

```cpp
static const double sunRiseAndSetTime = HOUR / 2; // 30 minutes
```

so `SUNRISE + sunRiseAndSetTime` = 6:30 AM, `SUNSET - sunRiseAndSetTime` = 5:30 PM, etc.

---

### lightColor

the color of the active light source (sun or moon). this is what actually tints shadows and direct illumination.

```cpp
static const double times[] = {
    MIDNIGHT,
    SUNRISE - HOUR,
    SUNRISE,
    SUNRISE + sunRiseAndSetTime / 4,
    SUNRISE + sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime / 2,
    SUNSET,
    SUNSET + HOUR/2,
    DAY
};
static const Color3 color[] = {
    Color3(.2, .2, .2),    // midnight — dim grey
    Color3(.1, .1, .1),    // pre-dawn — near black
    Color3(0, 0, 0),       // sunrise start — black
    Color3(.6, .6, 0),     // early sunrise — orange-yellow
    Color3(.75, .75, .75), // daytime — bright white (dayDiffuse)
    Color3(.75, .75, .75), // daytime — bright white
    Color3(.1, .1, .075),  // pre-sunset — faint warm
    Color3(.1, .05, .05),  // sunset — deep red-orange
    Color3(.1, .1, .1),    // post-sunset — dark
    Color3(.2, .2, .2)     // back to midnight
};
```

```cpp
void getLightColor(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::LightColor), out, sizeof(float) * 3, nullptr);
}

void setLightColor(uintptr_t base, float r, float g, float b) {
    float color[3] = { r, g, b };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LightColor), color, sizeof(float) * 3, nullptr);
}
```

---

### ambient

the general ambient light level — affects the base brightness of everything in the scene.

```cpp
static const double times[] = {
    MIDNIGHT,
    SUNRISE - HOUR,
    SUNRISE,
    SUNRISE + sunRiseAndSetTime / 4,
    SUNRISE + sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime / 2,
    SUNSET,
    SUNSET + HOUR/2,
    DAY
};
static const Color3 color[] = {
    Color3(0, .1, .3),     // midnight — deep blue
    Color3(0, .0, .1),     // pre-dawn — near black blue
    Color3(0, 0, 0),       // sunrise start — black
    Color3(0, 0, 0),       // early sunrise — still black
    Color3(.4, .4, .4),    // daytime — mid grey (dayAmbient = white * 0.4)
    Color3(.4, .4, .4),    // daytime
    Color3(.5, .2, .2),    // pre-sunset — warm pink
    Color3(.05, .05, .1),  // sunset — dark blue tint
    Color3(0, .0, .1),     // post-sunset — dark blue
    Color3(0, .1, .3)      // back to midnight
};
```

```cpp
void getAmbient(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Ambient), out, sizeof(float) * 3, nullptr);
}

void setAmbient(uintptr_t base, float r, float g, float b) {
    float color[3] = { r, g, b };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Ambient), color, sizeof(float) * 3, nullptr);
}
```

---

### diffuseAmbient

the diffuse component of ambient — softer than direct light but still contributes to scene brightness. peaks at full white during the day.

```cpp
static const double times[] = {
    MIDNIGHT,
    SUNRISE - HOUR,
    SUNRISE,
    SUNRISE + sunRiseAndSetTime / 2,
    SUNRISE + sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime / 2,
    SUNSET,
    SUNSET + HOUR/2,
    DAY
};
static const Color3 color[] = {
    Color3(.1, .1, .17),   // midnight — dim blue-grey
    Color3(.05, .06, .07), // pre-dawn — near black
    Color3(.08, .08, .01), // sunrise start — slight warm tint
    Color3(.75, .75, .75), // mid-sunrise — bright (white * 0.75)
    Color3(.75, .75, .75), // daytime — full bright
    Color3(.35, .35, .35), // late afternoon — dimming
    Color3(.5, .2, .2),    // pre-sunset — warm pink
    Color3(.05, .05, .1),  // sunset — dark blue
    Color3(.06, .06, .07), // post-sunset — near black
    Color3(.1, .1, .17)    // back to midnight
};
```

```cpp
void getDiffuseAmbient(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::DiffuseAmbient), out, sizeof(float) * 3, nullptr);
}

void setDiffuseAmbient(uintptr_t base, float r, float g, float b) {
    float color[3] = { r, g, b };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::DiffuseAmbient), color, sizeof(float) * 3, nullptr);
}
```

---

### skyAmbient

the sky's contribution to ambient — this is what gives the sky its color cast on surfaces. ramps up around sunrise, stays on through sunset, then cuts out.

```cpp
static const double times[] = {
    MIDNIGHT,
    SUNRISE - 2*HOUR,
    SUNRISE - HOUR,
    SUNRISE - HOUR/2,
    SUNRISE,
    SUNRISE + sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime,
    SUNSET,
    SUNSET + HOUR/3,
    DAY
};
static const Color3 color[] = {
    Color3(0, 0, 0),        // midnight — black
    Color3(0, 0, 0),        // pre-dawn — black
    Color3(.07, .07, .1),   // late pre-dawn — very dim blue
    Color3(.2, .15, .01),   // near sunrise — warm amber
    Color3(.2, .15, .01),   // sunrise — warm amber
    Color3(1, 1, 1),        // daytime — full white
    Color3(1, 1, 1),        // daytime
    Color3(.4, .2, .05),    // sunset — orange-brown
    Color3(0, 0, 0),        // post-sunset — black
    Color3(0, 0, 0)         // midnight
};
```

```cpp
void getSkyAmbient(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyAmbient), out, sizeof(float) * 3, nullptr);
}

void setSkyAmbient(uintptr_t base, float r, float g, float b) {
    float color[3] = { r, g, b };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyAmbient), color, sizeof(float) * 3, nullptr);
}
```

---

### skyAmbient2

a second sky ambient spline — this one has a wider ramp and contributes additional sky color. the difference between `skyAmbient` and `skyAmbient2` gives you the gradient from horizon to zenith.

```cpp
static const double times[] = {
    MIDNIGHT,
    SUNRISE - 3*HOUR,
    SUNRISE - 2*HOUR,
    SUNRISE - HOUR/2,
    SUNRISE,
    SUNRISE + sunRiseAndSetTime,
    SUNSET - sunRiseAndSetTime,
    SUNSET,
    SUNSET + HOUR/3,
    SUNSET + 2*HOUR,
    SUNSET + 3*HOUR,
    DAY
};
static const Color3 color[] = {
    Color3(0, 0, 0),            // midnight
    Color3(0, 0, 0) * 0.7,      // early pre-dawn — near black
    Color3(.3, .3, .4) * 0.7,   // pre-dawn — dim blue-grey
    Color3(.4, .3, .3),         // near sunrise — muted warm
    Color3(.3, .2, .3),         // sunrise — muted purple-pink
    Color3(1, 1, 1),            // daytime
    Color3(1, 1, 1),            // daytime
    Color3(.4, .3, .2),         // sunset — warm brown
    Color3(.3, .2, .3),         // post-sunset — muted purple
    Color3(.3, .2, .3),         // late post-sunset
    Color3(0, 0, 0),            // night fade
    Color3(0, 0, 0)             // midnight
};
```

```cpp
void getSkyAmbient2(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyAmbient2), out, sizeof(float) * 3, nullptr);
}

void setSkyAmbient2(uintptr_t base, float r, float g, float b) {
    float color[3] = { r, g, b };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyAmbient2), color, sizeof(float) * 3, nullptr);
}
```

> **the difference between skyAmbient and skyAmbient2:** `skyAmbient2` has a longer ramp and more keyframes, so it bleeds further into the night on both ends. in the original source these are referred to as `SkyColor` / `SkyColor2` or `SkyGradientTop` / `SkyGradientBottom` depending on context — both pairs mean the same thing. `skyAmbient` drives the bottom of the gradient (warmer near the horizon) and `skyAmbient2` drives the top (cooler/darker at zenith).

---

## Brightness

a single float in `[0.1, 1.0]` that scales overall scene brightness. peaks at noon (`1.0`), bottoms out at midnight (`0.1`). derived directly from the sun's y position, which is itself just a function of clock time.

```cpp
float getBrightness(uintptr_t base) {
    float brightness;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::Brightness), &brightness, sizeof(float), nullptr);
    return brightness;
}

void setBrightness(uintptr_t base, float brightness) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Brightness), &brightness, sizeof(float), nullptr);
}
```

---

## LightDirection

the direction vector pointing toward the active light source. in physical mode this is `trueSunPosition` during the day and `trueMoonPosition` at night. in basic mode it follows the same logic but uses the unrotated positions.

```cpp
void getLightDirection(uintptr_t base, float* out) {
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::LightDirection), out, sizeof(float) * 3, nullptr);
}

void setLightDirection(uintptr_t base, float x, float y, float z) {
    float dir[3] = { x, y, z };
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LightDirection), dir, sizeof(float) * 3, nullptr);
}
```

---

## ActiveSource

an integer flag indicating which light source is currently active. `0` = sun, `1` = moon. determined by whether the sun is above the `-0.3` y threshold in physical mode, or by angle in basic mode.

```cpp
int getActiveSource(uintptr_t base) {
    int source;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::ActiveSource), &source, sizeof(int), nullptr);
    return source; // 0 = sun, 1 = moon
}

void setActiveSource(uintptr_t base, int source) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ActiveSource), &source, sizeof(int), nullptr);
}
```

---

## Replicating it yourself (writing to offsets)

this is the part most people actually want. there's no single ClockTime param you can just write to — to replicate lighting at a given time you have to **derive every property from the time yourself** using the formulas, then write each one to its own offset separately. sun direction, moon direction, brightness, all five color channels — all of it, individually.

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
    uintptr_t LightColor      = 0x???;  // Color3 — lightColor spline result
    uintptr_t LightDirection  = 0x???;  // Vector3 — direction toward active light source
    uintptr_t Ambient         = 0x???;  // Color3 — ambient spline result
    uintptr_t DiffuseAmbient  = 0x???;  // Color3 — diffuseAmbient spline result
    uintptr_t SkyAmbient      = 0x???;  // Color3 — skyAmbient spline result (gradient bottom / SkyColor)
    uintptr_t SkyAmbient2     = 0x???;  // Color3 — skyAmbient2 spline result (gradient top / SkyColor2)
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

    float halfYear = 182.6282f;
    float tilt     = -0.4101f * cosf(M_PI * (dayOfYear - halfYear) / halfYear);
    float latRad   = geoLatDeg * (M_PI / 180.0f);
    ApplyTiltRotation(out, bx, by, bz, tilt - latRad);
}

void CalcTrueMoonDir(float* out, float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f, float moonPhase = 0.0f) {
    double angle = 2.0 * M_PI * (clockTime / 24.0);
    float curMoonPhase = moonPhase * (float)M_PI * 2.0f;
    float bx = (float)sin(curMoonPhase + angle);
    float by = (float)-cos(curMoonPhase + angle);
    float bz = 0.0f;

    float halfYear  = 182.6282f;
    float earthTilt = 0.4101f;
    float moonTilt  = 0.0873f; // ~5 degrees
    float tilt      = (-earthTilt + moonTilt) * sinf(moonPhase * 4.0f);
    float latRad    = geoLatDeg * (M_PI / 180.0f);
    ApplyTiltRotation(out, bx, by, bz, tilt - latRad);
}

bool CalcIsDaytime(float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    float sun[3];
    CalcTrueSunDir(sun, clockTime, geoLatDeg, dayOfYear);
    return sun[1] > -0.3f;
}

// lerp between two float[3] colors by t (0-1)
void LerpColor(float* out, float* a, float* b, float t) {
    out[0] = a[0] + (b[0] - a[0]) * t;
    out[1] = a[1] + (b[1] - a[1]) * t;
    out[2] = a[2] + (b[2] - a[2]) * t;
}

// linearly interpolate across a set of time/color keyframes
// times[] and colors[] must have `count` entries, times in hours (0-24)
void LinearSplineColor(float* out, float clockTime, float* times, float colors[][3], int count) {
    if (clockTime <= times[0])          { out[0]=colors[0][0]; out[1]=colors[0][1]; out[2]=colors[0][2]; return; }
    if (clockTime >= times[count - 1])  { out[0]=colors[count-1][0]; out[1]=colors[count-1][1]; out[2]=colors[count-1][2]; return; }
    for (int i = 0; i < count - 1; i++) {
        if (clockTime >= times[i] && clockTime < times[i + 1]) {
            float t = (clockTime - times[i]) / (times[i + 1] - times[i]);
            LerpColor(out, colors[i], colors[i + 1], t);
            return;
        }
    }
}

// returns lightColor (color of the active light source)
void CalcLightColor(float* out, float clockTime) {
    float times[]     = { 0, 5, 6, 6.125f, 6.5f, 17.5f, 17.75f, 18, 18.5f, 24 };
    float colors[][3] = {
        {.2f,.2f,.2f}, {.1f,.1f,.1f}, {0,0,0}, {.6f,.6f,0},
        {.75f,.75f,.75f}, {.75f,.75f,.75f}, {.1f,.1f,.075f},
        {.1f,.05f,.05f}, {.1f,.1f,.1f}, {.2f,.2f,.2f}
    };
    LinearSplineColor(out, clockTime, times, colors, 10);
}

// returns ambient
void CalcAmbient(float* out, float clockTime) {
    float times[]     = { 0, 5, 6, 6.125f, 6.5f, 17.5f, 17.75f, 18, 18.5f, 24 };
    float colors[][3] = {
        {0,.1f,.3f}, {0,0,.1f}, {0,0,0}, {0,0,0},
        {.4f,.4f,.4f}, {.4f,.4f,.4f}, {.5f,.2f,.2f},
        {.05f,.05f,.1f}, {0,0,.1f}, {0,.1f,.3f}
    };
    LinearSplineColor(out, clockTime, times, colors, 10);
}

// returns diffuseAmbient
void CalcDiffuseAmbient(float* out, float clockTime) {
    float times[]     = { 0, 5, 6, 6.25f, 6.5f, 17.5f, 17.75f, 18, 18.5f, 24 };
    float colors[][3] = {
        {.1f,.1f,.17f}, {.05f,.06f,.07f}, {.08f,.08f,.01f}, {.75f,.75f,.75f},
        {.75f,.75f,.75f}, {.35f,.35f,.35f}, {.5f,.2f,.2f},
        {.05f,.05f,.1f}, {.06f,.06f,.07f}, {.1f,.1f,.17f}
    };
    LinearSplineColor(out, clockTime, times, colors, 10);
}

// returns skyAmbient (sky gradient bottom / SkyColor)
void CalcSkyAmbient(float* out, float clockTime) {
    float times[]     = { 0, 4, 5, 5.5f, 6, 6.5f, 17.5f, 18, 18.333f, 24 };
    float colors[][3] = {
        {0,0,0}, {0,0,0}, {.07f,.07f,.1f}, {.2f,.15f,.01f},
        {.2f,.15f,.01f}, {1,1,1}, {1,1,1},
        {.4f,.2f,.05f}, {0,0,0}, {0,0,0}
    };
    LinearSplineColor(out, clockTime, times, colors, 10);
}

// returns skyAmbient2 (sky gradient top / SkyColor2)
void CalcSkyAmbient2(float* out, float clockTime) {
    float times[]     = { 0, 3, 4, 5.5f, 6, 6.5f, 17.5f, 18, 18.333f, 20, 21, 24 };
    float colors[][3] = {
        {0,0,0}, {0,0,0}, {.21f,.21f,.28f}, {.4f,.3f,.3f},
        {.3f,.2f,.3f}, {1,1,1}, {1,1,1}, {.4f,.3f,.2f},
        {.3f,.2f,.3f}, {.3f,.2f,.3f}, {0,0,0}, {0,0,0}
    };
    LinearSplineColor(out, clockTime, times, colors, 12);
}

// brightness peaks at noon (1.0), dips at night (0.1)
float CalcBrightness(float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    float sun[3];
    CalcTrueSunDir(sun, clockTime, geoLatDeg, dayOfYear);
    float t = (sun[1] + 1.0f) / 2.0f;
    return 0.1f + t * 0.9f;
}

// returns 0 (sun) or 1 (moon)
int CalcActiveSource(float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f) {
    return CalcIsDaytime(clockTime, geoLatDeg, dayOfYear) ? 0 : 1;
}

// returns the direction toward the active light source
void CalcLightDirection(float* out, float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f, float moonPhase = 0.0f) {
    float sunDir[3], moonDir[3];
    CalcTrueSunDir(sunDir, clockTime, geoLatDeg, dayOfYear);
    CalcTrueMoonDir(moonDir, clockTime, geoLatDeg, dayOfYear, moonPhase);
    bool daytime = CalcIsDaytime(clockTime, geoLatDeg, dayOfYear);
    float* src = daytime ? sunDir : moonDir;
    out[0] = src[0]; out[1] = src[1]; out[2] = src[2];
}
```

### writing everything to the game

```cpp
void SetLightingFromTime(uintptr_t base, float clockTime, float geoLatDeg = 0.0f, float dayOfYear = 0.0f, float moonPhase = 0.0f) {
    float sunDir[3], moonDir[3], lightDir[3];
    float lightColor[3], ambient[3], diffuseAmbient[3], skyAmbient[3], skyAmbient2[3];

    CalcTrueSunDir(sunDir,        clockTime, geoLatDeg, dayOfYear);
    CalcTrueMoonDir(moonDir,      clockTime, geoLatDeg, dayOfYear, moonPhase);
    CalcLightDirection(lightDir,  clockTime, geoLatDeg, dayOfYear, moonPhase);
    CalcLightColor(lightColor,    clockTime);
    CalcAmbient(ambient,          clockTime);
    CalcDiffuseAmbient(diffuseAmbient, clockTime);
    CalcSkyAmbient(skyAmbient,    clockTime);
    CalcSkyAmbient2(skyAmbient2,  clockTime);

    float brightness = CalcBrightness(clockTime, geoLatDeg, dayOfYear);
    int   source     = CalcActiveSource(clockTime, geoLatDeg, dayOfYear);

    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueSunDir),     sunDir,        sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::TrueMoonDir),    moonDir,       sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LightColor),     lightColor,    sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::LightDirection), lightDir,      sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Ambient),        ambient,       sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::DiffuseAmbient), diffuseAmbient,sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyAmbient),     skyAmbient,    sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::SkyAmbient2),    skyAmbient2,   sizeof(float)*3, nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::Brightness),     &brightness,   sizeof(float),   nullptr);
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::ActiveSource),   &source,       sizeof(int),     nullptr);
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

| Time (hrs) | Sun y  | Day/Night | Brightness | lightColor      | skyAmbient      |
|------------|--------|-----------|------------|-----------------|-----------------|
| `0` / `24` | `-1.0` | night     | `0.10`     | dark grey       | black           |
| `6.0`      | `0.0`  | threshold | `0.55`     | black           | warm amber      |
| `6.5`      | `+0.13`| day       | `0.61`     | bright white    | full white      |
| `12.0`     | `+1.0` | day       | `1.00`     | bright white    | full white      |
| `17.5`     | `+0.13`| day       | `0.61`     | bright white    | full white      |
| `18.0`     | `0.0`  | threshold | `0.55`     | deep red-orange | orange-brown    |
| `18.5`     | `-0.13`| night     | `0.49`     | dark            | black           |

the key thing to understand is that **every visual property is derived from the same angle** — `2π × (clockTime / 24)`. once you have that angle, everything else follows from it.

---

## Constants worth knowing

```cpp
const double HOUR          = 3600;
const double DAY           = 86400;
const double SIDEREAL_DAY  = 86164.1;  // used for star rotation
const double SUNRISE       = 6  * HOUR;
const double SUNSET        = 18 * HOUR;
const double NOON          = 12 * HOUR;
const double MIDNIGHT      = 0;

// sun/moon tilt constants
const double earthTilt     = toRadians(23.5);  // 0.4101 rad
const double moonTilt      = toRadians(5.0);   // 0.0873 rad

// moon phase
const double moonPhaseInterval = DAY * 29.53;
const double initialMoonPhase  = 0.75; // phase offset at Jan 1 1970

// star field
const double initialStarRot = 1; // approx offset at Jan 1 1970
```

---

*more stuff coming as i figure it out. PRs welcome if you spot something wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
