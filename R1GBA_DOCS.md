# R1-GameboyAdvance — Technical Documentation

## How It Works

EmulatorJS (mGBA WASM core) runs GBA games at full speed on the Rabbit R1's WebView. The R1 has no physical game buttons, so controls are mapped to:

- **PTT (side button)**: Single click = A, double click = B, long press = Start (hold 2s = exit to menu)
- **Scroll wheel**: Scroll down = D-pad RIGHT, Scroll up = D-pad LEFT
- **Touch zones**: Left third = LEFT, right third = RIGHT, top third = UP, bottom third = DOWN

## EmulatorJS Control Input

The key discovery: EmulatorJS does NOT respond to synthetic `KeyboardEvent`s reliably on the R1. Instead, we use `gameManager.simulateInput(player, button, value)` directly, which calls the WASM `simulate_input` function.

**Button indices** (from `initControlVars`):
| Index | Button |
|-------|--------|
| 0 | B |
| 1 | Y |
| 2 | SELECT |
| 3 | START |
| 4 | UP |
| 5 | DOWN |
| 6 | LEFT |
| 7 | RIGHT |
| 8 | A |
| 9 | X |
| 10 | L |
| 11 | R |

**Usage**: `gm.simulateInput(0, 8, 1)` = player 0, button A, pressed. `gm.simulateInput(0, 8, 0)` = released.

## Getting gameManager

`EJS_ready` fires 20ms after page load — too early, `gameManager` doesn't exist yet. Use `EJS_onGameStart` instead, which fires after the ROM loads and game starts. Even then, poll for `gameManager` with `setInterval` because timing can vary on the R1.

```javascript
EJS_onGameStart = function() {
  var tryGet = setInterval(function() {
    if (window.EJS_emulator && window.EJS_emulator.gameManager) {
      gm = window.EJS_emulator.gameManager;
      clearInterval(tryGet);
    }
  }, 300);
};
```

## Keyboard Event Fallback

If `simulateInput` fails, we fall back to dispatching `KeyboardEvent`s with **numeric keyCodes** (not `key` strings). EmulatorJS's `keyChange` method compares `this.controls[i][j].value === e.keyCode`, where values are numeric after `setupKeys()` converts strings via `keyMap`.

| Key | keyCode |
|-----|---------|
| z | 90 |
| x | 88 |
| Enter | 13 |
| Shift | 16 |
| ArrowLeft | 37 |
| ArrowUp | 38 |
| ArrowRight | 39 |
| ArrowDown | 40 |

## Screen Rotation

R1 screen is 240×282px portrait. GBA is 240×160 landscape. We rotate the container -90° to display in landscape:

```css
#game {
  width: 282px!important;
  height: 240px!important;
  left: -21px;
  top: 0;
  transform: rotate(-90deg);
  transform-origin: center center;
}
```

Touch zones are mapped to **visual directions** (not game coordinates) so the D-pad feels natural in landscape:
- Physical left → game LEFT
- Physical right → game RIGHT
- Physical top → game UP
- Physical bottom → game DOWN

## Hiding Virtual Gamepad

EmulatorJS shows an on-screen gamepad by default on touch devices. To hide it:
1. `EJS_VirtualGamepadSettings = []` — removes gamepad button definitions
2. CSS `[class*="virtual"],[class*="gamepad"]` — hides any remaining elements
3. `setInterval` DOM sweep — catches elements added after load

## R1 Event API

```javascript
window.addEventListener('sideClick', fn);       // PTT button
window.addEventListener('longPressStart', fn);    // long press begins
window.addEventListener('longPressEnd', fn);      // long press released
window.addEventListener('scrollUp', fn);          // scroll wheel up
window.addEventListener('scrollDown', fn);        // scroll wheel down
```

R1 fires `scrollUp` when scrolling up physically, `scrollDown` when scrolling down.

## Files

- `index.html` — Game menu with 32 games, redirects to `play.html?rom=<url>`
- `play.html` — EmulatorJS player with R1 control mapping
- `install.html` — QR code installation page

## Deployment

Push to `https://github.com/Aavirash/r1-GameboyAdvance.git` → GitHub Pages serves at `https://aavirash.github.io/r1-GameboyAdvance/`. The R1 caches by install URL, so bump `?v=N` in `install.html` to force updates.