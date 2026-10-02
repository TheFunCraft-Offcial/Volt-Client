# VoltClient

A client-side PvP HUD and quality-of-life client for **Minecraft 26.2** (Fabric).
56 modules, a draggable click GUI, and a snap-to-grid HUD editor.

Client-side only. Nothing here touches packets, reach, aim or timing — it is a
HUD/QoL client in the Lunar/Badlion mould, not a cheat client, so it is safe to
run on servers that allow standard client mods.

---

## Building

```bash
./gradlew build
```

The jar lands in `build/libs/voltclient-1.0.0.jar`. Drop it in `.minecraft/mods`
alongside Fabric API.

**Requirements:** JDK 25, Fabric Loader 0.19.3+, Fabric API, Minecraft 26.2.

### Note on the build script

26.x ships **unobfuscated**, which changes the buildscript in ways that trip
people up coming from 1.21.x:

| Old (1.21.x and earlier) | New (26.x) |
|---|---|
| `fabric-loom` / `fabric-loom-remap` | `net.fabricmc.fabric-loom` (non-remapping) |
| `modImplementation`, `modCompileOnly` | `implementation`, `compileOnly` |
| `remapJar` | `jar` |
| `mappings "net.fabricmc:yarn:..."` | *(no mappings block — Yarn is discontinued)* |
| `archivesBaseName = ...` | `base { archivesName = ... }` |
| `sourceCompatibility` at project level | inside the `java { }` block |

Gradle 9.6.1 is pinned in the wrapper because Gradle 8.x tops out at Java 24 and
26.2 needs 25.

---

## Controls

| Key | Action |
|---|---|
| `Right Shift` | Open the click GUI |
| `Right Ctrl` | Open the HUD editor |
| `C` | Zoom (hold) |
| `Left Alt` | Freelook (hold) |

All rebindable in Options → Controls. Individual modules can also be bound to a
key by **right-clicking** them in the click GUI.

**Click GUI:** left-click a module to toggle, click the arrow to expand its
settings, drag a panel header to move it, right-click a header to collapse.

**HUD editor:** drag elements to move, scroll over one to resize, middle-click
to reset it, `R` to reset everything. Elements snap to screen edges, centre
lines and to each other — hold `Alt` to place freely.

---

## Modules

### Combat (18)
CPS Counter · Keystrokes · Potion Timers · Durability · Damage Numbers ·
Kill Feed · Combo Counter · Target HUD · Attack Indicator · Totem Counter ·
Gear Overlay · Crit Flash · Nearby Alert · Combat Tag · Range Indicator ·
Shield Status · Sweep Counter · Durability Alert

### Movement (11)
Zoom · Freelook · Toggle Sprint · Toggle Sneak · Speed · Cooldowns ·
Elytra HUD · Safe Fall · Sprint Reset · Auto-Jump Status · Jump Height

### Cosmetic (10)
Theme · Hit Sounds · Health Tint · Death Summary · Fullbright ·
Custom Crosshair · Particle Reducer · HUD Frame · Screenshot Capture ·
Held Item

### Utility (17)
FPS · Ping · Coordinates · FPS Graph · Ping Graph · Session Stats ·
Chunk Borders · Toasts · Tab List · Clock · Waypoints · Mob Radar ·
Light Warning · Pickup Log · Inventory Full · Quick Chat · Performance Summary

Settings and HUD positions persist to `config/voltclient/config.json`.
HUD positions are stored as a fraction of the screen, so your layout survives
window resizes and GUI-scale changes.

---

## Architecture

```
VoltClient            entrypoint, wiring, screen helpers
├── compat/Render     ← every draw call in the mod goes through here
├── module/           Module base, settings types, ModuleManager
├── hud/              HudModule base, HUD layer, drag-and-drop editor
├── gui/              click GUI, Mod Menu hook
├── modules/          the 56 modules, one file each
├── config/           JSON persistence
└── mixin/            4 mixins: attack, mouse clicks, movement toggles, freelook camera
```

Two things worth knowing if you extend it:

**`compat/Render.java` is a deliberate firewall.** Minecraft's GUI layer has
been churning hard — `GuiGraphics` became `GuiGraphicsExtractor`, `drawString`
became `text`, the HUD matrix stack went from `PoseStack` to a 2D
`Matrix3x2fStack`, and `Screen#render` became `Screen#extractRenderState`.
Nothing outside `Render.java` touches those APIs. When the next version breaks
something, you fix one file and all 56 modules keep compiling.

**Modules are fail-soft.** `ModuleManager.safely()` catches anything a module
throws, disables that module, and logs it once. One bad module can't crash you
out mid-fight.

**Prefer polling a stable getter over hooking a method that keeps moving.**
`hurt()`/`die()` don't exist as declared methods anywhere in the
`LocalPlayer`/`Player` hierarchy in 26.2 (Mojang restructured damage handling
around a `ServerLevel`-taking `damage(...)` method instead), so
`ModuleManager` detects both by diffing `getHealth()` every tick instead of
injecting into either — the same technique `DamageNumbers` already used for
tracking other entities. `Zoom` follows the same principle: it drives the
public `Minecraft.options.fov()` setting directly rather than injecting into
`GameRenderer`'s internal FOV calculation, which is still being restructured
post-Vulkan and changes shape across 26.x point releases.

---

## Status

This compiles clean and has been confirmed launching successfully with mixins
applying correctly (`MultiPlayerGameModeMixin`, `MouseHandlerMixin`,
`LocalPlayerMixin` — the module count in the log should read "56 modules
loaded"). A few things changed shape from the original design during that
process, in the interest of trading a bit of polish for not depending on parts
of the 26.x API that are still actively being restructured:

- **Floating damage numbers** render as a HUD-anchored rising/fading column
  rather than hovering in 3D space over the target. True world-space
  projection needs the camera's view/projection matrices, which move around
  release to release right now (Vulkan backend transition) — not worth it for
  a cosmetic effect.
- **`TimeDisplay`'s in-game day counter was dropped.** `ClientLevel#getDayTime()`
  doesn't exist on this classpath and I don't have a confirmed replacement;
  the real-world clock is unaffected.
- **Freelook was rewritten properly, not just re-fixed.** The original just
  switched to third-person, which does nothing — mouse movement rotates the
  entity itself regardless of camera mode, so it never actually decoupled
  anything. It now freezes the real rotation tick-side (for movement/attacks)
  while a separate free angle drives the camera via a new `EntityMixin` on
  `getViewYRot`/`getViewXRot` — untested against a real launch like every
  mixin here, but the design itself is sound this time, not just the API
  calls.
- **Zoom drives `Minecraft.options.fov()` directly** instead of injecting into
  `GameRenderer`, since that internal method is exactly the part of the
  render pipeline still being restructured post-Vulkan. `Fullbright` and
  `Particle Reducer` (in the batch below) use the same trick on their own
  vanilla options.

A second batch of 23 modules was added on top of the original 33. A few of
those lean on the same "prefer polling a stable getter over guessing a
volatile one" principle — `Pickup Log` diffs inventory contents instead of
hooking a pickup event, `Combat Tag`/`Sweep Counter` reuse the existing
attack/health hooks rather than adding new ones. `Waypoints` and `Quick Chat`
are intentionally scoped down (in-memory only, preset messages only) to avoid
needing a text-input widget the click GUI doesn't have yet. `Screenshot
Capture` is the one module in the whole project calling an API I couldn't
verify at all (`net.minecraft.client.Screenshot`) — if that's wrong, it'll
disable itself via the usual fail-soft path rather than taking anything else
down.

If something is still visibly wrong in-game (a module drawing garbage, a
setting not doing what its name says), that's the kind of thing that only
shows up by actually playing with it — send me what you're seeing.

Separately, three modules (`ChunkBorders`, `HealthTint`, `TabListInfo`) are
**model-only**: they own their settings and expose the values, but the
world-render and tab-overlay mixins that would actually consume them were
never written, so they'll show up in the GUI without visibly doing anything.
Everything else is wired end to end.

## Renaming

```bash
grep -rl 'voltclient\|VoltClient' . | xargs sed -i 's/voltclient/yourname/g; s/VoltClient/YourName/g'
mv src/main/java/com/thefuncraft/voltclient src/main/java/com/thefuncraft/yourname
mv src/main/resources/assets/voltclient src/main/resources/assets/yourname
```

## Licence

MIT.
