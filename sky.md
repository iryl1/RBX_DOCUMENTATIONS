# Roblox Sky System Docs

sky is a pretty small class but there's a couple weird things in it i didn't expect — mainly that the front face setter has a GA call buried in it, and that `CelestialBodiesShown` is wired up differently from everything else.

---

## What this covers

- defaults — what the sky looks like before you touch anything
- the six skybox faces and how they're stored
- star count — clamping and defaults
- celestial bodies — why it behaves differently from the other properties
- the GA event hidden in the front face setter
- **how to read/write all of this yourself via offsets**

---

## Defaults

when a `Sky` is created it sets itself up like this:

```cpp
Sky::Sky()
    : drawCelestialBodies(true)
    , numStars(3000)
{
    setName("Sky");
    skyUp = ContentId::fromAssets("textures/sky/sky512_up.tex");
    skyLf = ContentId::fromAssets("textures/sky/sky512_lf.tex");
    skyRt = ContentId::fromAssets("textures/sky/sky512_rt.tex");
    skyBk = ContentId::fromAssets("textures/sky/sky512_bk.tex");
    skyFt = ContentId::fromAssets("textures/sky/sky512_ft.tex");
    skyDn = ContentId::fromAssets("textures/sky/sky512_dn.tex");
}
```

the default skybox is the classic roblox blue sky, 3000 stars, celestial bodies on. if you read the face offsets on a fresh Sky object before any game code runs, those asset paths are what you'll find.

---

## Skybox Faces

six faces, six `TextureId` fields. the naming is pretty intuitive once you know the abbreviations:

```
SkyboxUp  — top    (+Y)
SkyboxDn  — bottom (-Y)
SkyboxLf  — left   (-X)
SkyboxRt  — right  (+X)
SkyboxBk  — back   (-Z)
SkyboxFt  — front  (+Z)
```

five of the six setters are exactly the same — compare, write, raise property changed:

```cpp
void Sky::setSkyboxBk(const TextureId& texId)
{
    if (texId != skyBk)
    {
        skyBk = texId;
        raisePropertyChanged(prop_SkyBk);
    }
}
```

the front face (`SkyboxFt`) is different. it does all of the above but also has a GA tracking call tacked on:

```cpp
void Sky::setSkyboxFt(const TextureId& texId)
{
    if (texId != skyFt)
    {
        skyFt = texId;
        raisePropertyChanged(prop_SkyFt);

        static boost::once_flag flag = BOOST_ONCE_INIT;
        boost::call_once(flag, &sendSkyBoxStats, texId);
    }
}
```

`sendSkyBoxStats` sends a `"SkyBox"` event to GA with the asset id. the `boost::once_flag` means this fires exactly once per process — first game to set a custom front face gets logged, then the flag is gone. setting the front face to the same texture it already has won't trigger it either since the change check comes first.

### reading TextureId

`TextureId` is just a wrapper around `std::string`. the default paths (`textures/sky/sky512_ft.tex` etc.) are 26 chars which puts them over MSVC's SSO limit (~15 chars), so the actual string data is heap-allocated. to read a face texture you need to follow the string's internal pointer to get the content. for shorter custom asset ids (`rbxassetid://12345` is 19 chars) same deal — still heap.

---

## StarCount

clamped to `[0, 5000]`, default `3000`:

```cpp
void Sky::setNumStars(int value)
{
    value = std::min(value, 5000);
    value = std::max(value, 0);
    if (value != numStars)
    {
        numStars = value;
        raisePropertyChanged(prop_StarCount);
    }
}
```

writing directly to the offset skips the clamp. the change check also means if you write the same value that's already there, nothing happens — no property change, no re-render.

```cpp
int getNumStars(uintptr_t base) {
    int value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::NumStars), &value, sizeof(int), nullptr);
    return value;
}

void setNumStars(uintptr_t base, int value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::NumStars), &value, sizeof(int), nullptr);
}
```

---

## CelestialBodiesShown

controls whether the sun and moon render. defaults to `true`.

this one is wired up as a `BoundProp` instead of going through a getter/setter like everything else. `BoundProp` just binds directly to the field address — the reflection system reads and writes it without any function call in between. the practical difference is that **no property change event fires when this changes**. every other property on Sky calls `raisePropertyChanged` in its setter. this one never does. the renderer will read whatever's in the field, it just won't be notified that it changed.

```cpp
bool getCelestialBodiesShown(uintptr_t base) {
    bool value;
    ReadProcessMemory(hProcess, (LPVOID)(base + Offsets::DrawCelestialBodies), &value, sizeof(bool), nullptr);
    return value;
}

void setCelestialBodiesShown(uintptr_t base, bool value) {
    WriteProcessMemory(hProcess, (LPVOID)(base + Offsets::DrawCelestialBodies), &value, sizeof(bool), nullptr);
}
```

---

## Offsets

```cpp
namespace Offsets {
    // Sky instance base
    uintptr_t SkyUp               = 0x???;  // TextureId (std::string — follow heap ptr)
    uintptr_t SkyLf               = 0x???;  // TextureId
    uintptr_t SkyRt               = 0x???;  // TextureId
    uintptr_t SkyBk               = 0x???;  // TextureId
    uintptr_t SkyFt               = 0x???;  // TextureId — first change fires a one-time GA event
    uintptr_t SkyDn               = 0x???;  // TextureId
    uintptr_t DrawCelestialBodies = 0x???;  // bool (default true, no change event on write)
    uintptr_t NumStars            = 0x???;  // int (default 3000, setter clamps to [0, 5000])
}
```



---

*more stuff coming. PRs welcome if something's wrong.*

> 💬 discord: [discord.gg/JpeFatN8yn](https://discord.gg/JpeFatN8yn)
