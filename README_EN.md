# 🚀 Fix for Unclickable Bottom Screen Area (Dead Zone / Phantom Window) on Ubuntu/GNOME

This repository contains a technical guide and patch to fix the **unclickable bottom screen area** issue when using the **Dash2Dock Animated / Dash2Dock Lite** extension on GNOME Shell, especially under **Wayland** sessions on Ubuntu.

---

# 🐛 Problem Description

If you are using the **Dash2Dock Animated** extension, you may notice that the bottom area of the screen (right above the dock) suddenly becomes:

- unclickable
- blocked by an invisible transparent layer
- unable to interact with apps behind it

This issue is commonly known as:

- Dead Zone
- Phantom Window
- Invisible Hitbox
- Ghost Input Region

---

## Symptoms

- App buttons near the dock do not respond to clicks
- Browser or editor buttons like VSCode cannot be clicked
- The empty area above the dock feels blocked by something invisible
- The issue disappears when hover/magnify animation is disabled

---

# 🔍 Technical Root Cause

This issue is caused by a **reactive input region** or **transparent bounding box** created by the dock extension to detect mouse hover.

Inside `dock.js`, the extension calculates a footprint (`fp`) value using:

```javascript
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

This `fp` value is not only used for animation.

It is also used as:

- dock actor height
- mouse collision area
- reactive input region
- GNOME Shell allocation box

As a result:

```text
Small Visual Dock
But Extremely Large Hitbox
```

This transparent region blocks mouse interaction with applications behind it.

---

# ⚠️ Why Is This Worse on Wayland?

Under Wayland:

- input regions are stricter
- every surface strictly owns its input events
- transparent actors still receive mouse input

As a result, the invisible hitbox fully captures mouse click events.

Meanwhile on X11, event forwarding is more permissive, so the bug is sometimes less noticeable.

---

# 🛠️ Patch Solution

## Step 1 — Open the Extension Directory

```bash
cd ~/.local/share/gnome-shell/extensions/dash2dock-lite@icedman.github.com
```

---

## Step 2 — Edit `dock.js`

```bash
code dock.js
```

or:

```bash
nano dock.js
```

---

## Step 3 — Fix the Footprint (`fp`) Value

Find:

```javascript
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

Replace with:

```javascript
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify));
```

---

# 🧠 Why Does This Fix the Bug?

Previously:

```text
fp too large
↓
Dock actor becomes too tall
↓
Invisible hitbox expands upward
↓
Mouse clicks get blocked
```

After reducing it:

```text
Hitbox matches dock size
↓
Transparent area shrinks
↓
Mouse clicks work normally again
```

---

# ⚠️ New Problem: Bounce / Magnify Animation Gets Clipped

After reducing `fp`, a side effect usually appears:

- bounce animation gets clipped
- magnify effect gets cut off
- animation cannot overflow outside the dock

This happens because of:

```javascript
clip_to_allocation: true,
```

inside the main dock constructor.

---

# ✅ Additional Important Fix

Find:

```javascript
clip_to_allocation: true,
```

Replace it with:

```javascript
clip_to_allocation: false,
```

---

# 🔥 Why Is This Important?

With `clip_to_allocation: false`:

- icons can render outside the container
- bounce animation no longer gets clipped
- magnify becomes smooth again
- the dock still keeps a small hitbox
- dead zones remain fixed

---

# ✅ Best Patch Combination

## Patch 1 — Reduce Footprint

```javascript
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify));
```

## Patch 2 — Disable Clipping

```javascript
clip_to_allocation: false,
```

---

# 🎯 Final Result

✅ Dead zones removed  
✅ Bottom screen area becomes clickable again  
✅ Hover animation remains smooth  
✅ Bounce animation no longer clipped  
✅ Magnify works correctly  
✅ No more invisible phantom window  

---

# 🔄 Reload GNOME Shell

## X11 Users

```text
Alt + F2
r
Enter
```

## Wayland Users

```text
Log Out → Log In again
```

---

# 🧪 Experiment Tips

Try adjusting:

```javascript
iconSize * 0.8
iconSize * 0.9
iconSize * 1.0
iconSize * 1.1
```

The most stable values are usually around:

```text
0.9 to 1.1
```

---

# 🏷️ Keywords

`gnome-shell-extension`  
`dash-to-dock-animated`  
`dash2dock-lite`  
`wayland-bug`  
`ubuntu-unclickable-screen`  
`phantom-window`  
`hitbox-fix`  
`javascript-patch`  
`clip_to_allocation`
