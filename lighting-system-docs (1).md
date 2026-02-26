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

this one actually accounts for:
- earth's 23.5° axial tilt
- what time of year it is (seasons)
- your geographic latitude

the sun declination formula ends up being: `offset = -23.5° × cos(π × (day - 182.6) / 182.6) - latitude`

if you're just making a normal game you probably don't need this. but it's there.

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
// moonPhaseInterval = 29.53 days in seconds = 2,549,520
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

## Putting it all together — simple implementation

if you want to replicate the basics yourself:

```cpp
class SimpleLighting {
private:
    double timeInSeconds = 50400; // 2:00 PM by default

public:
    std::string getTimeString() {
        int hours   = (int)(timeInSeconds / 3600) % 24;
        int minutes = (int)(timeInSeconds / 60) % 60;
        int seconds = (int)timeInSeconds % 60;
        return std::to_string(hours) + ":" +
               std::to_string(minutes) + ":" +
               std::to_string(seconds);
    }

    void setMinutesAfterMidnight(double minutes) {
        timeInSeconds = fmod(minutes * 60, 86400);
    }

    Vector3 getSunDirection() {
        double angle = 2 * M_PI * timeInSeconds / 86400;
        return Vector3(sin(angle), -cos(angle), 0);
    }

    Vector3 getMoonDirection() {
        double angle = 2 * M_PI * timeInSeconds / 86400 + M_PI;
        return Vector3(sin(angle), -cos(angle), 0);
    }

    bool isDaytime() {
        return getSunDirection().y > 0;
    }
};
```

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

## Sky color throughout the day

the game uses spline interpolation to smoothly blend between these:

| Time     | Color (RGB)           | vibe            |
|----------|-----------------------|-----------------|
| Midnight | `(0.20, 0.20, 0.20)` | dark grey       |
| Sunrise  | `(0.60, 0.60, 0.00)` | orange          |
| Noon     | `(0.75, 0.75, 0.75)` | bright white    |
| Sunset   | `(0.10, 0.05, 0.05)` | deep red-orange |

---

## Usage examples

```cpp
lighting->setTimeStr("14:30:00");
lighting->setMinutesAfterMidnight(870); // same thing, 2:30 PM

double  minutes   = lighting->getMinutesAfterMidnight();
Vector3 sunDir    = lighting->getSunPosition();
float   moonPhase = lighting->getMoonPhase();
```

---

*more stuff coming as i figure it out. PRs welcome if you spot something wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
