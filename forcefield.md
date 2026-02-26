# Roblox ForceField System Docs

 ForceField is a lot simpler than the previous systems but there's some interesting stuff in how the detection and rendering work that's worth documenting properly.


---

## What this covers

- what ForceField actually is and how it works
- how `partInForceField` detection works (the traversal logic)
- the animation cycle system (legacy vs cyclic executive)
- render internals — the two-sphere effect, colors, sizes
- where ForceField can and can't be parented
- what's relevant for cheat dev

---

## What ForceField is

ForceField is an `Instance` that, when placed inside a character (or any ancestor of a part), makes that character immune to damage from `TakeDamage`. it renders as an animated double-sphere visual effect around the Torso. it's creatable from Lua and inherits from both `IAdornable` (for rendering) and `Effect`.

it **cannot** be parented directly to Workspace — `askSetParent` rejects it:

```cpp
bool ForceField::askSetParent(const Instance* instance) const {
    return (!Instance::fastDynamicCast<Workspace>(instance));
}
```

so it has to live inside a character model or one of its descendants.

---

## How damage immunity works

this is the part most people care about. the check used by `Humanoid::takeDamage` is:

```cpp
bool ForceField::partInForceField(PartInstance* part)
```

which calls `ancestorContainsForceField(part)` — a recursive upward traversal:

```cpp
bool ancestorContainsForceField(Instance* instance) {
    if (containsForceField(instance))   // check this node's direct children
        return true;

    if (Instance* parent = instance->getParent()) {
        if (!Instance::fastDynamicCast<Workspace>(parent))  // stop at Workspace
            return ancestorContainsForceField(parent);
    }
    return false;
}
```

and `containsForceField` just iterates the children of a node looking for a ForceField:

```cpp
bool containsForceField(Instance* instance) {
    for (size_t i = 0; i < instance->numChildren(); ++i) {
        if (Instance::fastDynamicCast<ForceField>(instance->getChild(i)))
            return true;
    }
    return false;
}
```

so the check starts at the part (e.g. Torso), looks at its direct children, then moves up to the parent, then its parent, and so on — stopping when it hits Workspace. this means a ForceField anywhere in the ancestor chain between the part and Workspace will block damage.

in practice this means:
- ForceField inside the character model → immune
- ForceField inside the Humanoid → immune
- ForceField directly on the part → immune
- ForceField anywhere else above it before Workspace → also immune

---

## Animation cycle system

ForceField has two animation modes depending on whether the TaskScheduler is running in cyclic executive mode or not.

**legacy mode** (non-cyclic executive):

```cpp
// constructor:
invertCycle = cycles() / 2;  // starts at 30 (halfway through a 60-step cycle)

// per render frame:
cycle       = (cycle + 1) % cycles();        // 0..59, loops
invertCycle = (invertCycle + 1) % cycles();  // same but offset by 30
```

**cyclic executive mode** (newer):

```cpp
// constructor:
startTime = Time::now<Time::Fast>();

// per render frame:
double offset = (now - startTime).seconds() * 60.0;
offset -= floor(offset);  // fractional part only, keeps it 0..1

int cycle       = (int)(offset * cycles());
int invertCycle = cycle + cycles() / 2;
if (invertCycle > cycles())
    invertCycle -= cycles();
```

both produce the same result — a 60-step animation cycle with the inverted version offset by 30 steps — just one is frame-step based and the other is time-based.

`cycles()` is a static method that always returns `60`.

---

## Render internals

the visual effect is two concentric spheres drawn around the Torso, both animating in and out but offset from each other.

```cpp
static const float largeSize = 1.1f;  // outer sphere scale multiplier
```

the animation value for each sphere is computed from the cycle:

```cpp
int max  = ForceField::cycles();  // 60
int half = max / 2;               // 30

// inner sphere: ramps up 0→1 then back down 1→0 over 60 steps
int value   = cycle < half ? cycle : max - cycle;
float percent = (float)value / (float)half;

// outer sphere: uses a different ramp with full-cycle denominator
int invertVal    = invertCycle < half ? invertCycle : max - invertCycle;
float invertPercent = (float)invertVal / (float)max;
```

then the two spheres are drawn:

```cpp
// position is set to the Torso's world position
adorn->setObjectToWorldMatrix(pos);

// inner sphere — solid-ish blue, pulsing opacity
adorn->sphere(
    Sphere(Vector3::zero(), part->getPartSizeXml().magnitude()),
    Color4(0, 51.0f/255.0f, 204.0f/255.0f, percent * 0.6f) * 0.8f
);

// outer sphere — cyan-to-blue shifting color, different opacity curve
adorn->sphere(
    Sphere(Vector3::zero(), part->getPartSizeXml().magnitude() * largeSize),  // 1.1x bigger
    Color4(0, invertPercent, 1.0f - invertPercent, invertPercent * 0.45f) * 0.8f
);
```

in plain terms:
- **inner sphere** — fixed blue `(0, 0.2, 0.8)`, radius = part bounding sphere magnitude, opacity pulses 0→0.48→0 over 60 frames
- **outer sphere** — color shifts between `(0, 0, 1)` and `(0, 1, 0)` (blue to cyan), radius = inner × 1.1, opacity also pulses but on the inverted cycle

there's a transparency guard — parts too close to the camera (`localTransparencyModifier >= 0.99`) skip rendering entirely, so you don't get the effect clipping through the camera.

**note:** when `RenderNewParticles2Enable` is true, `render3dAdorn` returns immediately and nothing is drawn. the visual effect is handled by the particle system instead in those builds.

the Torso pointer is cached on first render by searching the parent's children for `"Torso"`:

```cpp
if (!torso.get())
    torso = shared_from(Instance::fastDynamicCast<PartInstance>(
        this->getParent()->findFirstChildByName2("Torso", true).get()
    ));
```

this only works for R6 since it hardcodes `"Torso"`. for R15 you'd need `"UpperTorso"` instead — the source doesn't handle this.

---

## What's relevant for cheat dev

ForceField is mostly relevant as a **damage immunity check** rather than something you write offsets to. the key points:

**bypassing the check** — `takeDamage` calls `partInForceField(torso)`. if you want to bypass ForceField protection to deal damage directly, write health via its offset instead of calling `takeDamage`. the ForceField check only lives in `takeDamage`, not in the raw health setter.

**faking a ForceField** — you don't need to write any offsets. just parent a ForceField instance to the target character in Lua (if you have the ability to run scripts on the server). the detection is purely instance-tree based.

**detecting if a player has a ForceField** — traverse their character model looking for a ForceField child anywhere in the tree, same as the source does. there's no cached flag or bitmask to read — it's always a live traversal.

**the animation data** — `cycle` and `invertCycle` are just cosmetic and have no effect on the damage immunity. you don't need to care about them for any gameplay purpose.

---

## Constants

```cpp
static int cycles() { return 60; }   // animation cycle length
static const float largeSize = 1.1f; // outer sphere scale vs inner
```

---

*more systems coming. lmk if something's off.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
