# Rescue Rangers Mobile Touch Controls -- Design Document

## 1. Overview

This document describes the design for adding mobile touch controls to the Rescue Rangers HTML5 helicopter game, optimizing for iPad landscape play. The goal is to replicate all keyboard functionality through on-screen touch controls while preserving the desktop keyboard experience unchanged.

**Target device**: iPad (large tablet) in landscape orientation  
**Reference illustration**: `RR_MOBILE_RESCUE_RANGER_ONSCREEN_CONTROLS_03.png`  
**Original specification**: `initial_spec_for_RR_Mobile_variant.md`

---

## 2. Library Selection: nipplejs

**Choice: [nipplejs](https://github.com/yoannmoinet/nipplejs) v0.10.2 via CDN**

| Criteria | nipplejs | Custom implementation | virtualjoystick.js |
|----------|----------|----------------------|-------------------|
| Dependencies | Zero | Zero | Zero |
| Size | ~9KB min | ~3KB | ~5KB |
| Multi-touch | Yes | Must build | Limited |
| iOS Safari tested | Yes | Must test | Dated |
| Dead zone support | Built-in | Must build | No |
| Maintenance | Active | N/A | Abandoned |
| CDN available | Yes | N/A | No |

**Why nipplejs wins**: It handles all the hard edge cases (touch drift, multi-touch conflicts, iOS-specific quirks) out of the box. A custom implementation would add 100+ lines of touch math for the same result.

**CDN source**: `https://cdn.jsdelivr.net/npm/nipplejs@0.10.2/dist/nipplejs.min.js`

**Buttons**: No library needed. Custom HTML `<button>` elements with `touchstart`/`touchend` event handlers. Simpler, more control, and trivial to implement.

---

## 3. Architecture: How Touch Integrates with the Game

### The key insight

The game already uses a `keys = {}` object that's polled every frame in `update()`. Touch controls simply set/unset entries in this object. **Zero changes to game physics, weapons, or AI logic.**

### Integration patterns by control type

| Control | Key simulated | Pattern | Why |
|---------|--------------|---------|-----|
| Joystick | w, a, s, d | Continuous set/unset based on angle | Movement is polled every frame |
| Shoot | space | Hold = key down, release = key up | Firing is polled with cooldown |
| Missile | m | Tap = press+release cycle | Uses `_mslCd` cooldown guard |
| Bomb | b | Tap = press+release cycle | Uses `_bombCd` cooldown guard |
| Drop troops | e | Hold = key down, release = key up | Uses `_dropCd` guard, hold to drop multiple |
| Pickup | f | Tap fires directly | One-shot action in keydown handler |
| Buy units 1-4 | 1, 2, 3, 4 | Tap calls buy logic directly | One-shot action in keydown handler |
| Pause | escape | Tap toggles G.state | State toggle |
| Shop | tab | Tap toggles G.showShop | UI toggle |

### Multi-touch safety

Each button tracks its `Touch.identifier` to prevent cross-contamination. If a player holds Shoot with one finger and taps Missile with another, releasing Missile won't accidentally release Shoot.

---

## 4. Control Layout

### Default layout (right-handed)

```
+------------------------------------------------------------------+
|  $1620            [HELICOPTER HP]  [ENEMY BASE HP]                |
|  AMMO: 45/70     [YOUR BASE HP]                                  |
|  MISSILES: 2                                                     |
|                                                    [S] [M] [1][2]|
|     +-------+                                      [E] [F] [3][4]|
|     |       |                                                     |
|     |  (o)  |    <-- joystick                                     |
|     |       |                                                     |
|     +-------+                                                     |
|  [PAUSE][SHOP]                                              [B]  |
|  [BOMB: 10]                                                       |
|========= BOMBS ===== TROOPS ===== LIVES: 3 =====================|
+------------------------------------------------------------------+
```

### Left side (joystick hand)
- **Joystick zone**: ~35vmin square, positioned at left: 2vw, top: 15vh
- **PAUSE button**: Below joystick, left-aligned
- **SHOP button**: Next to PAUSE
- **HUD info**: Money, ammo, missiles displayed above joystick (HTML overlay, updated from game loop)

### Right side (action hand)
- **2x4 button grid**: Positioned at right: 2vw, top: 8vh
  - Row 1: **S** (Shoot), **M** (Missile), **1** (Infantry), **2** (Tank)
  - Row 2: **E** (Drop), **F** (Pickup), **3** (AA), **4** (Van)
- **B** (Bomb): Toggle button, positioned below the grid or near bottom-right
- Action buttons (S, M, E, F, B) styled with green borders
- Deploy buttons (1, 2, 3, 4) styled with yellow borders (matching current HUD colors)

### Button sizing
- Diameter: **13vmin** (~56px on iPad landscape)
- Exceeds Apple HIG minimum of 44pt tap targets
- Gap: 1.2vmin between buttons
- Total grid width: ~60vmin, fits comfortably in half the screen

### Visual style
- Semi-transparent backgrounds: `rgba(0, 255, 0, 0.12)` for action, `rgba(255, 255, 0, 0.1)` for deploy
- Bright borders: `#0f0` green for action, `#ff0` yellow for deploy
- Pressed state: increased opacity (`0.35`) and lighter border
- Monospace font matching the game's retro aesthetic
- Deploy buttons dim when player can't afford the unit

---

## 5. Joystick Configuration

```javascript
nipplejs.create({
  zone: document.getElementById('tc-joystick-zone'),
  mode: 'static',         // Fixed position (always visible)
  position: { left: '50%', top: '50%' },
  color: '#0f0',           // Green to match game aesthetic
  size: 120,               // Base diameter
  threshold: 0.15,         // 15% dead zone
  fadeTime: 100,
  restOpacity: 0.6
});
```

### Angle-to-key mapping

The joystick provides angle (radians) and force (0-1). We map to 8-directional input:

- Divide the circle into 8 sectors (45 degrees each)
- Use `cos(angle)` and `sin(angle)` with a threshold of 0.38 (~67.5 degrees)
- This allows clean diagonals: pushing upper-right sets both `keys['w']` and `keys['d']`
- Force below 0.15 = dead zone = all keys released

The game's existing physics (`HELI_ACCEL = 0.35`, `HELI_DAMP = 0.94`) provide smooth acceleration/deceleration, so binary key states from the joystick feel natural.

---

## 6. Lefty / Righty Hand Swap

### Mechanism
- A CSS class `.lefty` on `#touch-controls` reverses all `left`/`right` CSS properties
- Joystick moves to right side, buttons move to left side
- Button grid order mirrors so action buttons (S, M) stay closest to the thumb near the joystick
- The nipplejs instance is destroyed and recreated in the new zone

### Access
- Toggle available in the pause screen
- Small gear icon in the control area
- Preference stored in `localStorage.setItem('rr-hand-pref', 'left'|'right')`
- Applied on page load

---

## 7. HUD Modifications

### What changes on mobile

| Element | Desktop | Mobile |
|---------|---------|--------|
| Keyboard hints (top-right) | "WASD:Fly SPACE:Shoot..." | **Hidden** |
| Money/Ammo/Missiles (top-left) | Canvas-drawn | **HTML overlay** (repositioned to avoid joystick overlap) |
| Bottom bar buy prices | "[1] Infantry $50..." | **Hidden** (buttons show this) |
| Bottom bar BOMBS/TROOPS/LIVES | Shown | **Kept** (important status) |
| Shop overlay "Press TAB" | Shown | Changed to "Tap SHOP to close" |
| Briefing "Press SPACE" | Shown | Replaced with **TAP TO START** button |
| Defeat "Press R" | Shown | Replaced with **TAP TO RESTART** button |

### HUD sync function
A lightweight `updateTouchHUD()` function called from `render()` updates the HTML overlay text. Uses change detection to avoid DOM thrashing:

```javascript
let _lastMoney = -1, _lastAmmo = -1, _lastMsl = -1;
function updateTouchHUD() {
  if (!isMobile) return;
  if (G.money !== _lastMoney) { moneyEl.textContent = '$' + G.money; _lastMoney = G.money; }
  // ... same for ammo, missiles
}
```

---

## 8. Game State Handling

Touch overlay buttons appear/disappear based on `G.state`:

| Game State | Joystick | Action Buttons | State Button |
|------------|----------|---------------|--------------|
| `briefing` | Hidden | Hidden | TAP TO START |
| `playing` | Visible | Visible | None |
| `paused` | Hidden | Hidden | TAP TO RESUME |
| `defeated` | Hidden | Hidden | TAP TO RETRY |
| `victory` | Hidden | Hidden | TAP TO RESTART |
| `missionComplete` | Hidden | Hidden | NEXT MISSION |

A `updateTouchVisibility()` function runs each frame to manage this.

---

## 9. Mobile Detection & Browser Setup

### Detection
```javascript
const isMobile = ('ontouchstart' in window) || 
                 (navigator.maxTouchPoints > 0) ||
                 (window.matchMedia('(pointer: coarse)').matches);
```

### Viewport meta (updated)
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, 
  maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
```

### PWA-like fullscreen on iPad
```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

### Prevent browser gestures
```css
html, body, canvas {
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
  overscroll-behavior: none;
}
```

### Safe areas (iPad notch/home indicator)
```css
#touch-controls {
  padding: env(safe-area-inset-top) env(safe-area-inset-right) 
           env(safe-area-inset-bottom) env(safe-area-inset-left);
}
```

---

## 10. Performance Considerations

1. **DOM overlay, not canvas-drawn**: Touch controls are HTML elements, zero cost to game rendering
2. **Change detection for HUD updates**: Only touch DOM when values actually change
3. **`will-change: transform`** on buttons for GPU compositing
4. **`passive: false`** on touch listeners to allow `preventDefault()` (prevents scroll)
5. **No continuous timers**: Joystick uses nipplejs events, buttons use native touch events
6. **Orientation change**: Destroy and recreate nipplejs instance rather than continuous resize polling

---

## 11. File Organization

**Single-file architecture maintained.** The game remains one HTML file with embedded CSS and JS.

Additions organized as clearly marked sections:

```
<head>
  <style>
    /* existing styles */
    /* ====== MOBILE TOUCH CONTROLS CSS ====== */
    ...
  </style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>
  
  <!-- ====== MOBILE TOUCH CONTROLS HTML ====== -->
  <div id="touch-controls">...</div>
  
  <script src="nipplejs CDN"></script>
  <script>
    // existing game code (~1928 lines)
    
    // ====== MOBILE TOUCH CONTROLS JS ======
    // ~250-350 lines
  </script>
</body>
```

Total file size: ~2300-2400 lines. Still manageable, no build tools needed, matches existing project pattern.

---

## 12. Implementation Phases

### Phase 1: Foundation
- Update viewport meta tag
- Add touch-prevention CSS
- Add `isMobile` detection
- Add empty `#touch-controls` container

### Phase 2: Joystick
- Add nipplejs via CDN
- Create joystick zone with CSS
- Implement move/end event handlers
- **Milestone**: Helicopter flies with touch

### Phase 3: Action Buttons (Shoot, Missile, Bomb, Drop, Pickup)
- Add button HTML and CSS
- Implement touch handlers with proper key simulation
- Handle multi-touch identifiers
- **Milestone**: All weapons work via touch

### Phase 4: Deploy Buttons (1-4)
- Add numbered buttons with yellow styling
- Direct function calls for purchases
- Affordability dimming (update opacity per frame)
- **Milestone**: Unit purchasing works via touch

### Phase 5: Utility (Pause, Shop)
- Add PAUSE and SHOP buttons
- Toggle game state / shop overlay
- Make shop overlay touch-friendly
- **Milestone**: Full game navigation via touch

### Phase 6: HUD Cleanup
- Hide keyboard hints on mobile
- Add HTML overlay HUD with change detection
- Adjust bottom status bar
- **Milestone**: Clean, mobile-appropriate display

### Phase 7: Game State Overlays
- Add TAP TO START / RESTART / NEXT MISSION / RESUME buttons
- Visibility management per game state
- Hide controls during non-playing states
- **Milestone**: Complete game flow via touch

### Phase 8: Hand Swap
- Implement `.lefty` CSS class
- Add toggle in pause menu
- localStorage persistence
- **Milestone**: Left-handed players supported

### Phase 9: Polish
- Haptic feedback (`navigator.vibrate()`)
- Fine-tune sizing for iPad
- Orientation lock hint for portrait mode
- Multi-touch edge case testing
- **Milestone**: Ship-ready

---

## 13. Testing Checklist

- [ ] iPad Safari landscape: all controls functional
- [ ] Multi-touch: fly + shoot simultaneously
- [ ] Multi-touch: fly + bomb + shoot
- [ ] Desktop Chrome: no controls visible, keyboard works
- [ ] All 6 missions playable start to finish via touch
- [ ] Pause/resume cycle
- [ ] Shop open/close and purchase via touch
- [ ] Lefty/righty swap works and persists
- [ ] No browser gestures (scroll, zoom, back-swipe)
- [ ] Briefing/defeat/victory/missionComplete transitions
- [ ] Orientation change doesn't break controls
- [ ] Performance: steady 60fps on iPad

---

## Sources & References

- [nipplejs - GitHub](https://github.com/yoannmoinet/nipplejs)
- [MDN: Mobile touch controls for games](https://developer.mozilla.org/en-US/docs/Games/Techniques/Control_mechanisms/Mobile_touch)
- [Apple HIG: Game Controls](https://developer.apple.com/design/human-interface-guidelines/game-controls)
- [Gamepad Controller](https://github.com/rokobuljan/gamepad) (evaluated, not selected)
- [Touch Control Design - Mobile Free To Play](https://mobilefreetoplay.com/control-mechanics/)
- [Multi-touch game controller in JS/HTML5 for iPad](https://seblee.me/2011/04/multi-touch-game-controller-in-javascripthtml5-for-ipad/)
