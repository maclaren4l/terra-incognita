---
name: run
description: Launch Terra Incognita locally and drive it end-to-end (mark a country, submit a guess, verify scoring) via Safari + JavaScript injection. Use this whenever asked to run, test, or verify a change to index.html actually works in a browser.
---

# Running and testing Terra Incognita

This is a single static `index.html` (no build step). "Running" it means
serving the file and actually playing a round in a real browser — canvas
rendering and d3 interaction can't be verified by reading the source.

## 1. Serve it

```bash
cd "/Users/umairahmed/Documents/Claude Projects/terra-incognita"
python3 -m http.server 8934 &>/tmp/terra-server.log &
curl -sf http://localhost:8934/index.html -o /dev/null && echo up
```

Stop it later with `pkill -f "http.server 8934"`.

## 2. Open it in a *scriptable* Safari document — not the pinned web app

This machine has an existing Safari window for this site pinned as a
"web app" (added to Dock). That window shows up fine in screenshots and
even responds to real OS clicks (`System Events click at`), but it is
**not** enumerated by `tell application "Safari" to documents` — so
`do JavaScript ... in document` cannot target it, and there is no
per-country accessibility element to click via UI scripting.

Instead, open a fresh normal tab, which *is* a scriptable document:

```bash
osascript -e 'tell application "Safari" to make new document with properties {URL:"http://localhost:8934/index.html"}'
sleep 1.5
osascript -e 'tell application "Safari" to do JavaScript "document.title" in document 1'
# => "Terra Incognita — Name the Country"
```

If this errors about JavaScript from Apple Events being disabled, enable
it once via Safari's Develop menu ("Allow JavaScript from Apple Events") —
Develop menu must be turned on first in Safari > Settings > Advanced.

From here, `document 1` is the game tab for the rest of the session
(unless you open other tabs first — check with
`tell application "Safari" to name of every document`).

## 3. Don't click by guessing screen pixels — dispatch events with known coordinates

Two dead ends to skip:

- **`System Events click at {x,y}` (OS-level click) mostly doesn't work**
  for marking a country, even when the click lands on a confirmed land
  pixel (verified by sampling screenshot pixel colors). It reliably hits
  UI buttons (SPIN/STOP/difficulty), but not the d3 canvas's own
  click-to-mark logic.
- **`new MouseEvent('click', ...)` alone, dispatched via JS, also does
  nothing.** d3's drag/zoom behavior listens for **Pointer Events**, not
  plain mouse events. You must dispatch `pointerdown`/`pointerup` (plus
  the matching mouse events for good measure) for the canvas's handlers
  to fire.

What actually works — get the canvas's real backing-store bitmap instead
of reasoning about screen/CSS pixel scaling:

```javascript
// find land coordinates directly from the canvas's own pixels
document.querySelector('canvas').toDataURL('image/png')
```

Run that through `do JavaScript`, pipe the base64 payload to a file,
decode it (`base64` module in Python — `pip3 install --user Pillow` if
PIL isn't present), and eyeball or grid-scan it for a solid tan-colored
land blob (`r>140 and g>120 and b<150 and r>b+30` is a decent land-vs-navy-
ocean filter). The canvas is `devicePixelRatio`-scaled (typically 2x), so
divide the pixel coordinate by `devicePixelRatio` to get the CSS-pixel
offset, then add `canvas.getBoundingClientRect().left/top` to get page
coordinates for the synthetic event.

```javascript
(function(){
  var c = document.querySelector('canvas');
  var r = c.getBoundingClientRect();
  var x = r.left + PIXEL_X / 2;   // divide by devicePixelRatio
  var y = r.top + PIXEL_Y / 2;
  var opts = {clientX:x, clientY:y, bubbles:true, cancelable:true,
              view:window, pointerId:1, isPrimary:true, button:0,
              pointerType:'mouse'};
  c.dispatchEvent(new PointerEvent('pointerdown', opts));
  c.dispatchEvent(new MouseEvent('mousedown', opts));
  c.dispatchEvent(new PointerEvent('pointerup', opts));
  c.dispatchEvent(new MouseEvent('mouseup', opts));
  c.dispatchEvent(new MouseEvent('click', opts));
})();
```

**The globe takes two clicks**, exactly like a real user: while it's
auto-spinning, the first click just stops it (banner changes from "The
globe is turning — click it to stop." to "Click a country to mark it …").
The *second* click, landing on land, actually marks the country. Do the
stop-click on any point (ocean is fine) before the mark-click.

## 4. Submit a guess

The countdown timer is real wall-clock time (30s easy / medium, 10s hard)
— if you mark a country and then guess in a *separate* `do JavaScript`
call, the round-trip latency between tool calls can burn the timer and
you'll get a timeout instead of a real answer. Combine marking + typing +
submitting into **one** `do JavaScript` script so it all happens
synchronously in one shot:

```javascript
(function(){
  // ... dispatch the mark click as above ...
  var input = document.getElementById('guess');
  var setter = Object.getOwnPropertyDescriptor(
    window.HTMLInputElement.prototype, 'value').set;
  setter.call(input, 'China');               // must use the native setter —
  input.dispatchEvent(new Event('input', {bubbles:true})); // plain .value= won't fire React-style/vanilla JS listeners
  var btn = document.getElementById('nameBtn')
    || Array.from(document.querySelectorAll('button'))
         .find(b => /name it/i.test(b.textContent));
  btn.click();
  return document.body.innerText.slice(0, 500);  // read back score/round state
})();
```

Reading `document.body.innerText` after each step is the easiest way to
confirm what happened (score, rounds, accuracy, the "Correct —"/"Time's
up —"/"✗"/"✓" messages) without needing a screenshot.

Hard mode adds a second round-trip: a correct name guess puts the UI in a
"CAPITAL REQUIRED" state ("What's the capital of X?") — submit the
capital through the same `#guess` input / `#nameBtn` flow for full
credit.

## 5. Visual sanity check (optional but recommended for rendering changes)

For anything touching the CSS theme or canvas drawing, also grab a real
screenshot for a human-eyeball check:

```bash
osascript -e 'tell application "Safari" to activate'
sleep 1
screencapture -x /path/to/screenshot.png
```

Read the PNG with the Read tool to actually look at it — don't just
assume the JS-level checks cover visual regressions.

## 6. Clean up

```bash
osascript -e 'tell application "Safari" to close document 1'   # the scriptable test tab
pkill -f "http.server 8934"
```

Leave the user's pinned web-app window alone — it's their persistent
shortcut to the site, not a test artifact.
