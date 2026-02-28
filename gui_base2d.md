# Roblox GuiBase2d Docs

been digging through the GuiBase2d source and writing down how it actually works. covers the resize/layout pipeline, position/size storage, rendering traversal, and child validation. might save you some headaches if you're working with Roblox's GUI internals.

---

## What this covers

- how absolute position and size are stored (float vs rounded integer versions)
- the resize pipeline — who calls what and in what order
- how child resizing propagates
- the recursive render traversal
- child validation — what's allowed under a GuiBase2d
- the rect helpers and what they return

---

## Position and Size Storage

GuiBase2d stores position and size in **two separate pairs** of Vector2 fields:

```cpp
Vector2 absolutePosition;       // rounded to nearest integer
Vector2 absolutePositionFloat;  // exact float value

Vector2 absoluteSize;           // rounded
Vector2 absoluteSizeFloat;      // exact float
```

the float versions are the source of truth. the non-float versions are just `Math::roundVector2(value)` applied on top. this matters because layout math uses floats internally, but the public-facing `AbsolutePosition` and `AbsoluteSize` properties expose the rounded versions.

### setAbsolutePosition / setAbsoluteSize

```cpp
bool GuiBase2d::setAbsolutePosition(const Vector2& value, bool fireChangedEvent)
{
    if (absolutePositionFloat != value) {
        absolutePositionFloat = value;
        absolutePosition = Math::roundVector2(value);
        if (fireChangedEvent)
            raisePropertyChanged(prop_AbsolutePosition);
        return true;
    }
    return false;
}
```

both setters follow the same pattern:
- compare against the float version (not the rounded one) to detect actual change
- store both the float and rounded copies
- fire the property changed event if requested
- return `true` if the value actually changed, `false` if it was a no-op

the return value is important — the resize pipeline uses it to decide whether to propagate to children.

---

## Rect Helpers

two variants:

```cpp
Rect2D GuiBase2d::getRect2D() const {
    return Rect2D::xywh(absolutePosition, absoluteSize);        // rounded integers
}

Rect2D GuiBase2d::getRect2DFloat() const {
    return Rect2D::xywh(absolutePositionFloat, absoluteSizeFloat); // exact floats
}
```

`getRect2D()` is what's used for visibility checks (`isVisible` compares against this). `getRect2DFloat()` is what the resize pipeline uses when computing child rects — since `getChildRect2D()` defaults to returning `getRect2DFloat()`.

```cpp
virtual Rect2D getChildRect2D() const { return getRect2DFloat(); }
virtual Rect2D getCanvasRect()  const { return getChildRect2D(); }
```

subclasses can override `getChildRect2D()` to give children a different layout region than the element's own rect (e.g. for padding or clipping).

---

## The Resize Pipeline

this is the main thing to understand about GuiBase2d. resize flows top-down from a root caller.

### handleResize

```cpp
void GuiBase2d::handleResize(const Rect2D& viewport, bool force)
{
    if (recalculateAbsolutePlacement(viewport) || force) {
        visitChildren(boost::bind(&ResizeChildren, _1, getChildRect2D(), force));
    }
}
```

two things happen here:

1. `recalculateAbsolutePlacement(viewport)` is called on this element. if it returns `true` (position or size changed) **or** `force` is set, children get resized too.
2. children are resized using `getChildRect2D()` — not the original viewport. so each level in the tree receives its parent's child rect as its own viewport.

### recalculateAbsolutePlacement

the base implementation just applies the viewport directly:

```cpp
bool GuiBase2d::recalculateAbsolutePlacement(const Rect2D& viewport)
{
    bool result = false;
    result = setAbsolutePosition(viewport.x0y0());
    result = setAbsoluteSize(viewport.wh()) || result;
    return result;
}
```

subclasses (like Frame, ImageLabel, etc.) override this to do UDim2-based layout math before calling into the setters. the base class version is essentially "fill the viewport completely."

### ResizeChildren (the propagation callback)

```cpp
static void ResizeChildren(shared_ptr<RBX::Instance> instance, const Rect2D& viewport, bool force)
{
    if (RBX::GuiBase2d* guiBase = Instance::fastDynamicCast<RBX::GuiBase2d>(instance.get())) {
        guiBase->handleResize(viewport, force);
    } else if (RBX::Folder* f = Instance::fastDynamicCast<RBX::Folder>(instance.get())) {
        f->visitChildren(boost::bind(&ResizeChildren, _1, viewport, force));
    }
}
```

two cases:
- if the child is a `GuiBase2d`, call `handleResize` on it normally
- if the child is a `Folder`, skip the folder itself but still recurse into *its* children with the same viewport

this means Folders are transparent to the resize pipeline — they pass the viewport through unchanged. everything else that isn't a `GuiBase2d` just gets ignored entirely.

---

## Recursive Rendering

rendering is driven externally (by ScreenGui or similar) — `shouldRender2d()` returns `false` by default, meaning GuiBase2d elements don't self-register for rendering. instead, the root calls `recursiveRender2d` and it walks the tree manually.

```cpp
void GuiBase2d::recursiveRender2d(Adorn* adorn)
{
    render2d(adorn);
    visitChildren(boost::bind(&GuiBase2d::RecursiveRenderChildren, _1, adorn));
}

static void RecursiveRenderChildren(shared_ptr<RBX::Instance> instance, Adorn* adorn)
{
    if (RBX::GuiBase2d* guiBase = Instance::fastDynamicCast<RBX::GuiBase2d>(instance.get())) {
        guiBase->recursiveRender2d(adorn);
    }
}
```

order is: render self, then recurse into children. non-`GuiBase2d` children are silently skipped. unlike the resize pipeline, Folders are not given special treatment here — they just get skipped.

---

## Child Validation

```cpp
bool GuiBase2d::askAddChild(const Instance* instance) const
{
    return (Instance::fastDynamicCast<GuiBase2d>(instance) != NULL);
}
```

only `GuiBase2d` instances (or subclasses) can be parented under a `GuiBase2d`. anything else gets rejected. the exception is that `Folder` is explicitly handled in the *resize* pipeline even though it can't actually be added as a child through normal means — something to be aware of if you're tracing that code path.

---

## ZIndex and GuiQueue

both are stored on GuiBase2d and initialized in the constructor:

```cpp
GuiBase2d::GuiBase2d(const char* name)
    : Super(name)
    , zIndex(GuiBase::minZIndex2d())
    , guiQueue(GUIQUEUE_GENERAL)
{}
```

`zIndex` starts at the minimum 2D z-index. `guiQueue` defaults to `GUIQUEUE_GENERAL`. subclasses can call the protected `setGuiQueue(queue)` to change the queue. both values feed into `GuiBase` interface implementations:

```cpp
virtual int      getZIndex()    const { return zIndex; }
virtual GuiQueue getGuiQueue()  const { return guiQueue; }
```

---

## AbsolutePosition and AbsoluteSize as Properties

these are registered as read-only Lua-accessible properties:

```cpp
Reflection::PropDescriptor<GuiBase2d, Vector2> GuiBase2d::prop_AbsoluteSize(
    "AbsoluteSize", category_Data, &GuiBase2d::getAbsoluteSize, NULL, Reflection::PropertyDescriptor::UI);

Reflection::PropDescriptor<GuiBase2d, Vector2> GuiBase2d::prop_AbsolutePosition(
    "AbsolutePosition", category_Data, &GuiBase2d::getAbsolutePosition, NULL, Reflection::PropertyDescriptor::UI);
```

the `NULL` setter means they're read-only from Lua. they expose the *rounded* values (`absolutePosition` / `absoluteSize`), not the float ones. the float versions are internal only.

---

## Quick Reference — resize call chain

```
[root] handleResize(viewport, force)
  └─ recalculateAbsolutePlacement(viewport)   ← returns true if pos/size changed
       └─ setAbsolutePosition(...)            ← compares float, stores both, fires event
       └─ setAbsoluteSize(...)
  └─ [if changed || force] visitChildren → ResizeChildren(child, getChildRect2D(), force)
       └─ if GuiBase2d → child.handleResize(childRect2D, force)
       └─ if Folder    → recurse into folder's children with same childRect2D
       └─ else         → skip
```

---

## Things worth knowing

- the `force` flag bypasses the change check — useful if you need to guarantee children are resized even when the parent's position/size didn't actually change
- `getChildRect2D()` and `getCanvasRect()` are both virtual — override `getChildRect2D()` if you want children to lay out inside a different region than the element's own rect
- visibility check (`isVisible`) uses the *rounded* `getRect2D()`, not the float version
- rendering skips Folders; resizing does not (Folders are transparent to resize but opaque to render)
- `GuiBase2d` is `DescribedNonCreatable` — it's a base class, not directly instantiable from Lua

---

*more stuff as i dig through it. lmk if something's off.*
