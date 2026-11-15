# iPhone Duo design doc

How to make a web game (or any full-screen web app) work on the **iPhone Duo**: both screens, closed and open, in Safari and as a home-screen app. Written from the Callout Golems work (gamegolemstudios, `public/games/callout-golems`, the `[duo]` comments in `css/style.css` and `js/main.js`, and `dev/games/callout-golems/tests/duotest.js`). Every number here was measured from screenshots taken on the device, not from spec sheets.

> Status: version 2, from the final round that shipped as Callout Golems v300 (every seat checked at every Duo size by `duotest.js`). Open questions that need a real device are in section 10.

---

## 1. The device in numbers

| | Physical px | CSS px (DPR 3) | Notes |
|---|---|---|---|
| Inner (open) screen | 2007 x 2853 | **669 x 951** | Held sideways in the home-screen app, the layout viewport is about **866 x 587**; iOS keeps the status rail and a bottom strip outside the page. |
| Cover (closed) screen | 1398 x 2034 | **466 x 678** | 678 x 466 held sideways. |

- **Status rail.** A strip about **84 pt** wide runs down the camera edge. It holds the clock, wifi and battery, and it **never goes away**, not even in the home-screen app with `black-translucent`. `env(safe-area-inset-*)` reports it as about 84px on that side. Where it sits:
  - cover screen upright: on the right;
  - cover screen sideways, camera on the left: on the left (overpaint does fill this one);
  - inner screen sideways: on the right, about 85pt;
  - inner screen upright: across the top, about 81pt.
- **Bottom strip.** In the inner screen's home-screen app there is also a strip of about 82px along the bottom, outside the page.
- **Safari vs the app.** Safari can't give up the rail at all. Only the home-screen app runs under part of it.
- **Insets to fake in tests** `[top, right, bottom, left]`:
  - cover sideways: 84 on the rail side, 21 at the bottom;
  - cover upright: `[0, 84, 34, 0]`;
  - inner home-screen app: all 0.
- **Hole punch.** The camera hole is at the **top of the right edge** with the closed phone held upright, just above the clock/wifi block. The punch and the clock/wifi block together are the only area UI really must avoid. The rest of the rail is usable.
- **Turning the phone.** The punch and clock box stay in the same physical corner, so in CSS terms they move to another corner of the page. Use `screen.orientation.angle` (or `window.orientation` as a fallback) to tell which one.
- **Folding.** There is no fold API. A fold, unfold or turn arrives only as a `resize`, sometimes a beat late, so re-measure on `resize`, `orientationchange` and `visualViewport` `resize`, and again about 250 ms later.

## 2. Size classes to design for

| Class | Media query | Device case |
|---|---|---|
| Cover, sideways | `(max-height: 500px) and (orientation: landscape)` | 678 x 466 |
| Cover, upright | `(max-width: 560px) and (orientation: portrait)` | 466 x 678 |
| Mid (inner, sideways) | `(orientation: landscape) and (min-height: 501px) and (max-height: 700px)` | 866 x 587 |
| Phone portrait | `(max-width: 700px) and (orientation: portrait)` | Also covers the inner screen upright, 669 x 951 |
| Tablet | landscape, 701px+ tall | iPad; unchanged |

The rules that came out of it:

| Class | Layout |
|---|---|
| Cover, sideways | Slide menus. The vehicle button bar is one row at the bottom centre; the wheel is 132px; the dash sits at bottom 160; the hand brake at bottom 122. |
| Cover, upright | Slide menus. The vehicle button bar is 3 columns on the right, filled from the bottom up. The right column holds the job dock at 272, the seat keys at 322 and the phone button under the minimap. |
| Mid | Tablet layout. Go to and BOARD are anchored from the bottom beside the button bar, which keeps them clear even if the height is ±80px off. Job docks folded; pilot buttons 2x2. |

The **mid class matters**. The open phone sideways (866 x 587, or 951 x 669 in Safari) falls between phone and tablet layouts, and both overlap there. Callout Golems' dispatch desk also switches to its phone layout below 760 wide or 520 tall.

The cover screen held sideways is the hardest case: a phone's height with an 84px rail eating into the width. Design that first.

## 3. Full screen: what works and what doesn't

### Use these

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Your Game">
<meta name="theme-color" content="#000000">
```
In the manifest, set `"display": "standalone"` and `"theme_color": "#000000"`. `display_override` can still list `fullscreen` first for browsers that honour it.

Call `requestFullscreen` on the first tap on phone-sized touch screens, and ignore it if the browser refuses. iPhone Safari refuses; other browsers may not.

```css
html { background: #000; }           /* What WebKit paints past the page (the rail, the bottom strip): make it the bezel colour. */
body { background: var(--bg); }
canvas#cv { position: fixed; inset: 0; width: 100%; height: 100%; display: block; }
```

### Don't

- **Don't size the game with `100vh`, `100lvh` or `100dvh`.** In the home-screen app these came out about 80px **short** of the page, which left a bar along the bottom. Use `position: fixed; inset: 0` (or `100%` of a fixed parent).
- **Don't expect `theme-color` or `black-translucent` to remove the rail.** They only make it black.
- **Don't name a variable `screen`.** A `const screen = ...` helper shadowed `window.screen`, and an iPhone-only code path that read `screen.width` before that line threw a TDZ `ReferenceError` and **the game never loaded on the phone**. Desktop tests passed because the path was gated on the user agent. Always write `window.screen`.
- **Don't rely on the Fullscreen API.** iPhone Safari doesn't offer it. Call it only where it exists.

## 4. Overpaint: drawing into the rail

WebKit shows whatever lies past the page in the rail area. If the canvas extends past the page, the rail shows the game world instead of a black bar.

```css
html.bleed canvas#cv {
  inset: -120px auto auto -120px;
  width: calc(100% + 240px);
  height: calc(100% + 240px);   /* Size in % of the page, never vh; a canvas won't stretch between insets. */
}
```

- Turn it on only for the Duo: an iPhone UA with `min(screen.width, screen.height)` between 450 and 700 (not ordinary iPhones or iPads), or `?bleed=1`.
- The renderer has to compensate so the visible view is pixel-identical:
  1. Read the offset back from the canvas's bounding rect.
  2. Keep the logical width and height (W, H) as the page size.
  3. Scale the projection by W / (W + 2M), where M is the margin.
  4. Extend the sky or background to cover the full canvas.
  5. Map `project()` results back into page pixels so labels and hit tests stay on the page.
- **Result:** it fills the rail on the cover screen held sideways. On the cover screen upright and on the inner screen held sideways, iOS still owns the rail, so it stays black (bezel-coloured), which is acceptable.
- `?railtest=1` turns the root background magenta. Magenta in the rail means the page can paint it. Black or green means iOS owns it.

## 5. The keep-out box

Don't throw away the whole 84px rail. Keep UI out of **one padded box**: the punch plus the clock/wifi block.

- **Box:** 156px along the rail from the punch end, and rail width + 4px across.
- **Which end:** upright, it's the top of the right rail. Held sideways, it's whichever end `screen.orientation.angle` (90 / 270 / -90) puts the physical corner at.
- **Detect a rail:** any inset of 50px or more on the left or right is a rail. Smaller insets are ordinary notch or home-bar insets; treat them normally.
- **CSS variables** (set from JS on `:root`):
  - `--sat/--sar/--sab/--sal`: the HUD insets, which are **0 on the rail side**, so HUD can use the rail.
  - `--rsat/--rsar/--rsab/--rsal`: the full raw insets, for menus, dialogs and dense panels that should simply stay inside the safe area.
  - `--ctl/--ctr/--cbl/--cbr`: the box's width in whichever corner it's in, and 0 in the others. Example: `right: calc(8px + var(--sar) + var(--ctr, 0px))` keeps a top-right bar left of the box.
- **Nudge pass:** every 400 ms (and on resize), any top-level HUD element overlapping the box is moved sideways by the shorter way using the CSS `translate` property, so the element's own `transform` is kept. Touch-control layers are excluded; they keep the whole rail.
- **Lay out around it:** in each orientation, UI goes **left of the box and below it**, never under it.

Reference implementation: the `SA` block near the top of `js/main.js` in Callout Golems.

## 6. Fitting controls on the cover screen: slide menus

The cover screen held sideways (about 590 x 466 usable) can't fit a desktop-style button cluster. Keep the essentials out and fold the rest behind a **"⋯" button**:

- **Always visible:** the movement stick or wheel, the pedals, the main action (WORK / USE), enter/exit (GET IN), and any button that is currently lit or "ready".
- **Behind "⋯":** everything less used.
- **Interaction:** **press "⋯", slide onto a button and release** to use it in one motion, or **tap "⋯", then tap a button** as a fallback. Highlight the button under the finger while sliding.
- **Open direction:** sideways, the menu opens as a row to the left of "⋯"; upright, as a column above it. Nothing under the thumb moves when it opens.
- **Badge:** show how many buttons are hidden, e.g. "⋯ 9".
- **Minimum hit size:** 44 to 48 CSS px.
- **How it fires:**
  - Pressing, sliding and releasing on an item fires that item by sending it a synthetic pointerdown and pointerup.
  - A tap with no slide only opens the menu.
  - A real pointerdown anywhere else closes it at once. **Don't use timers** for this; long frames delay them.
- **Nothing moves under the thumb:** the bar is `display: flex; flex-wrap: wrap-reverse`, and the always-visible items plus "⋯" get `order: -1`. Opened items then go on above.
- **Hooks:** a `.fan-t` toggle and `.fan-open` / `.fan-hov` classes; containers are marked `[data-fan]`. Job docks use `.cmp`, `.fold` and `.open`, plus `[data-jmore]` for the "⋯ N" button. CSS shows or hides by `[data-t="X"]` key. The toggle is `display: none` outside the cover size classes, so larger screens are unchanged.
- **Long tool rows** (for example air traffic control's tools): show the first tool plus "⋯ N", and open the rest as a column of buttons at least 44pt tall. Don't let a row of truncated pills ("RAD…", "GRO…") cover the scene.

## 7. Per-size layout rules that came up

- **Top bar:** one line, `white-space: nowrap`, a `max-width` that subtracts the insets and the corner box. Lines that don't fit go into the menu, not onto a second overlapping line.
- **Notifications (radio log, toasts):** go **under** the top bar, using a measured top-bar height variable (`--tbh`), never on top of it.
- **Minimap column (right side, top down):** minimap, then the phone/app button under it, with chips such as REP **beside** the button, not under it.
- **Job card on the cover screen upright:** down the left side under the top bar, scrollable, with `max-height` clear of the bottom controls.
- **Steering wheel:** 132px, not 158px; 158px was a third of the cover screen's height.
- **Pilot-style button sets:** a 2x2 block, not a long row.
- **Dense dashboards** (truck dash): move to the top left when the cover screen is held sideways.
- **Title screen on a small screen:** put the text on a dark wash; aim the camera further down so a bright horizon isn't behind the logo.
- **iPad:** everything must still read well there. The size classes above leave iPad layouts unchanged.

## 8. Testing

Desktop tests miss most of this. Test like the phone:

1. **Use an iPhone user agent.** UA-gated code paths are where the fatal bug was.
2. **Fail the test on any page error** (`page.on('pageerror')`).
3. **Run every size:** 678x466, 466x678, 866x587 and 669x951, plus a normal phone and an iPad.
4. **Fake the insets:** `?sa=t,r,b,l` (CSS px) stands in for `env(safe-area-inset-*)`, e.g. `?sa=0,84,0,0` for the rail on the right.
5. **Calibrate the box:** `?cut=x,y,w,h` overrides the keep-out box.
6. **Overpaint:** use `?bleed=1` to force it on and `?bleed=0` to force it off.
7. **Overlap check:** collect the bounding rects of the visible HUD elements and assert that none overlap each other or the keep-out box, for every seat or screen in the game.
8. **Smoke test before every release:** load the page with the iPhone UA at 866x587 and 678x466 and confirm it reaches the title with no errors.
9. **Release:** bump the service worker cache version on every release. A network-first worker isn't what keeps an old layout on screen, because its cache only answers offline. iOS's HTTP cache or a page left open is the usual cause, so tell testers to close the app fully and reopen it.

Reference test: `dev/games/callout-golems/tests/duotest.js`.
- Playwright with the iPhone UA, `deviceScaleFactor: 3`, `isMobile` and `hasTouch`.
- For every view at every size it checks that:
  - the canvas covers the viewport;
  - nothing crosses the page edge, the faked safe area or the keep-out box;
  - no two controls overlap, and no HUD panel covers a control;
  - the slide menu opens and fires;
  - the overpaint projection lines up.
- Usage: `node dev/games/callout-golems/tests/duotest.js [view ...] [size:<name> ...]` with the repo served on port 8360. `OUT=` sets the screenshot folder; `NOSHOT=1` skips screenshots.
- Other flags: `?touch=1` forces the touch controls, and `?nosw=1` skips the service worker.
- iPhone UA used: `Mozilla/5.0 (iPhone; CPU iPhone OS 26_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/26.0 Mobile/15E148 Safari/604.1`.

## 9. Checklist for a new game

- [ ] `viewport-fit=cover` plus the Apple meta tags; manifest `display: standalone`
- [ ] `html { background: #000 }`; game surface `position: fixed; inset: 0`; no vh, lvh or dvh
- [ ] All safe-area use goes through `--sa*` / `--rsa*` variables set from JS
- [ ] Rail detection (inset of 50px or more) and the keep-out box following `screen.orientation.angle`
- [ ] Overpaint for canvas games, with the renderer compensating
- [ ] Cover-screen size classes with "⋯" slide menus for less-used controls
- [ ] Notifications under the top bar; nothing overlapping at any of the four sizes
- [ ] `window.screen`, never a bare `screen`
- [ ] iPhone-UA Playwright test that fails on page errors, at every size
- [ ] Service worker version bumped on release

## 10. Open questions (need a real device)

- **Overpaint reach:** it fills the rail on the cover screen held sideways. On the cover screen upright and the inner screen sideways, the rail looks owned by iOS. Confirm with `?railtest=1`.
- **Keep-out box position:** the box is estimated from where the clock sits. The inner screen's punch position is not measured yet. Calibrate it from a device screenshot with `?cut=x,y,w,h`.
- **Rotation corner:** which corner `screen.orientation.angle` picks on the cover screen hasn't been checked on a device.
- **Fullscreen:** whether the Duo's Safari supports `requestFullscreen` is unknown.
- **Very narrow Safari portrait (382px wide):** this is the densest layout. In Callout Golems, the truck cab's BOARD and phone buttons still overlap by 18px there.
- **Leftover vh:** some `max-height` rules still use vh. That's safe, because it can only come out shorter than the screen, but `%` is better.
