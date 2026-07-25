# Development Notes — Launchpad MK2 Component

Internals of `launchpad_mk2.tox`, ported from Owen Kirby's 2017 Launchpad **Pro** component.
For using the component, see [README.md](README.md).

**Read [Gotchas](#gotchas) before changing anything.** Most of the time spent on this port went
to seven non-obvious TouchDesigner behaviours, all documented there with their symptoms.

---

## Current state

| | |
|---|---|
| Component | `launchpad_mk2` (container COMP) |
| Panel size | 300 × 300, resizable — all geometry is expression-driven |
| Pages / banks | 4 — `launchpad_base0`–`launchpad_base3` (the original's banks 4–7 were deleted) |
| Buttons per bank | 80 visible of 100: 64 pads + 8 top row + 8 right column |
| Original | on git `master` as `launchpad_pro.tox` |

---

## Hardware reference

### Pro vs MK2

| | Launchpad Pro | Launchpad MK2 |
|---|---|---|
| SysEx device ID | `02 10` (dec 16) | `02 18` (**dec 24**) |
| Set-LED command | `0B` (dec 11) | same |
| RGB range | 0–63 per channel | same |
| 8×8 pads | notes 11–88 (`10×row + col`) | same |
| Top row | CC 91–98 | **CC 104–111** |
| Right column | CC 19–89 | **notes** 19–89 |
| Bottom row | CC 1–8 | *does not exist* |
| Left column | CC 10–80 | *does not exist* |
| Velocity / aftertouch | pressure sensitive | fixed 127, no aftertouch |

Full LED message: `F0 00 20 29 02 18 0B <led> <r> <g> <b> F7`, sent as

```python
op('midiout1').send(240, 0, 32, 41, 2, 24, 11, led, r, g, b, 247)
```

### MK2 top row identity

| Button | Hardware CC | TD channel | After `mk2_ccremap` | Internal button | LED |
|---|---|---|---|---|---|
| Up | 104 | `ch1ctrl105` | `ch1ctrl92` | `button92` | 104 |
| Down | 105 | `ch1ctrl106` | `ch1ctrl93` | `button93` | 105 |
| Left | 106 | `ch1ctrl107` | `ch1ctrl94` | `button94` | 106 |
| Right | 107 | `ch1ctrl108` | `ch1ctrl95` | `button95` | 107 |
| Session | 108 | `ch1ctrl109` | `ch1ctrl96` | `button96` | 108 |
| User 1 | 109 | `ch1ctrl110` | `ch1ctrl97` | `button97` | 109 |
| User 2 | 110 | `ch1ctrl111` | `ch1ctrl98` | `button98` | 110 |
| Mixer | 111 | `ch1ctrl112` | `ch1ctrl99` | `button99` | 111 |

The arrows (104–107) are the page keys. Session/User 1/User 2/Mixer stay available as ordinary
per-page buttons — they do send MIDI normally (verified with a logger), though the device also
acts on them internally to change its own layout.

---

## Numbering conventions

Three numbering schemes coexist. Confusing them is the biggest source of bugs here.

1. **Hardware address** — what the device sends, and what LEDs are addressed by (`11`–`88`, `104`–`111`).
2. **TD channel name** — TouchDesigner names MIDI channels **1-based**: hardware note 11 arrives
   as `ch1n12`, hardware CC 104 as `ch1ctrl105`. See [Gotcha 1](#1-touchdesigner-names-midi-channels-1-based).
3. **Internal button number** — `button1`…`button100`, matching the TD channel number. Grid
   position is `p = number - 1`, then `col = p % 10`, `row = p // 10`, with **row 0 at the bottom**.

`table2` (Table DAT at the component's top level) converts internal → hardware:

| key (`button number - 1`) | value (LED / hardware address) |
|---|---|
| `11` | `11` — pads are identity |
| `88` | `88` |
| `91`–`98` | `104`–`111` — the MK2 top row |

Only the top-row rows were remapped for MK2. Everything else is unchanged from the Pro.

---

## Signal flow

```
midiin1                     MIDI In CHOP, all channel-1 notes + CCs
  └─ mk2_ccremap            Rename CHOP: MK2 top row → the Pro numbering the rest expects
       ├─ mk2_pageselect    Select CHOP  ch1ctrl9[2-5]  → the four page keys
       │    └─ mk2_pagechange   CHOP Execute → sets par.Presets
       └─ mk2_pagefilter    Delete CHOP  removes ch1ctrl9[2-5] so pages aren't also buttons
            └─ launchpad_base0..3        one bank per page
                 └─ in1 → select3 → merge1 → reorder1 → limit2 → null9
                      └─ chopexec1       interactMouse() on buttonNN → panel state
                           └─ button_state (per bank)
                                └─ rename0..3   prefixes channels 1_ 2_ 3_ 4_
                                     └─ merge3 → out1
```

LED output is a separate branch: `switch1 → button_state → chopexec3_`, plus
`colour_change → chopexec5_` for animated colours.

---

## Nodes added by this port

All at the top level of `launchpad_mk2`.

| Node | Type | Purpose |
|---|---|---|
| `mk2_ccremap` | Rename CHOP | MK2 top row `ch1ctrl105–112` → `ch1ctrl92–99`. Also parks stale Pro-era `ch1ctrl92–99` as `x92–x99` to avoid duplicate channel names. |
| `mk2_pageselect` | Select CHOP | `ch1ctrl9[2-5]` — the four page keys. |
| `mk2_pagefilter` | Delete CHOP | Removes `ch1ctrl9[2-5]` from the bank feed so page keys are global only. |
| `mk2_pagechange` | CHOP Execute | Off-to-On → `parent().par.Presets = page`. Nothing else — the existing chain follows from it. |
| `mk2_pageleds` | CHOP Execute | Watches `select1`. Radio-lights the four page LEDs, then schedules `refresh_all()`. |
| `mk2_keepalive` | Execute DAT | On Frame Start. Keeps the MIDI chain cooking. **Essential** — see [Gotcha 2](#2-chop-execute-dats-do-not-keep-their-chop-cooking). |
| `mask` | Container COMP | Bare overlay (opaque black constant), placeholder for custom artwork. |

Modified from the original: `chopexec3_`, `chopexec5_`, `table2`, every bank's `select3`, and all
button geometry.

---

## Page (preset) system

**Which page is visible is decided by the `layer` parameter of the bank containers, not
`display`.** All banks are permanently `display = True`, stacked at x=0, y=0. The selected one
gets `layer = 1` and the rest `0`, so it draws on top. Nothing is hidden — it is pure z-ordering.

```
par.Presets (menu 1-4)
  → par1 (Parameter CHOP) → select1 → math1/math2 → null9
      → null9_export → index of switch1, switch2, switch4, switch5, switch7
          → switch7 (one-hot patterns from base2's out1..out8)
              → switch7_export → layer of launchpad_base0..3
```

- `base2` holds the eight one-hot layer patterns (`out1`–`out8`), one per preset.
- `switch1/2/4/5` route the selected bank's CHOP data (states, colours, velocity) to the outputs.

**Adding or removing a page touches three places:** the `Presets` menu, a bank container (plus a
`switch7_export` row for its `layer`), and a constant in `base2`. Also add the new bank's `null9`
to `mk2_keepalive`.

### Stale export tables

The `name` column in the export tables is wrong in places. `null9_export` asks for a channel
called `radio` but `null9`'s channel is actually `Presets`; `null1_export` asks for `Presets`
where `null1`'s is `chan1`. These work only because TouchDesigner resolves them by the **index**
column. If you edit those rows, keep the index correct and ignore the names.

`switch7_export` also contains four dead rows targeting operators named `base3` / `base6` — the
banks are `launchpad_base3` / `launchpad_base6`. They do nothing. Harmless leftovers from an
earlier naming scheme, and the reason every bank's `display` is a plain constant.

---

## LED output

One function decides colour: **`chopexec3_.paint(n, on=None)`**. It resolves the LED from
`table2`, reads the button's current state when the caller doesn't already know it, and picks the
matching colour table. Callers:

| Caller | Fires when |
|---|---|
| `chopexec3_.offToOn` / `onToOff` | a button's state changes (passes the known state) |
| `chopexec5_.valueChange` | a button's *colour* changes — animation, or a page switch |
| `mk2_pageleds` → `refresh_all()` | after a page change, via `run(..., delayFrames=2)` |

`refresh_all()` exists because a button whose state is identical on both pages fires no
transition, but still needs the new page's colour.

### Colour table naming is graphic-based, not state-based

```python
IDLE, ACTIVE = 'rgbon_chop', 'rgboff_chop'
```

A button at rest shows the `on` TOP; while **active** it shows the `off` TOP. Verified against
the on-screen switch: state 0 selects input 0 (`on`), state 1 selects input 1 (`off`).
**Do not "correct" this to the obvious pairing** — the LEDs would then disagree with the
on-screen buttons.

### Page LEDs are reserved

`PAGE_LEDS = (104, 105, 106, 107)` is guarded in **both** `chopexec3_` and `chopexec5_`.
`mk2_pageleds` owns those four exclusively; without the guard the generic per-button output
repaints them the instant a page changes and the radio state is lost.

`While On` is deliberately **off** on `chopexec3_`. It re-sends an unchanged colour every frame —
harmless while the branch was dormant, but a 60fps SysEx flood now that the keep-alive forces
cooking. Off-to-On and On-to-Off cover every transition.

Page-key colours are the two constants at the top of `mk2_pageleds`: `ON = (63,63,63)`,
`OFF = (2,2,2)`.

---

## Panel layout

A gapless **9 × 9** fill: the MK2's missing left column and bottom row are dropped, leaving 8 pad
columns + the right column, and 8 pad rows + the top row. The top-right cell has no button.

Geometry is expression-driven off `me.digits` and the parent size, so it rescales:

```python
x = round((((me.digits - 1) % 10)  - 1) * me.parent().par.w / 9)
w = round((((me.digits - 1) % 10)) * me.parent().par.w / 9) - x    # spans to the next boundary
y = round((((me.digits - 1) // 10) - 1) * me.parent().par.h / 9)
h = round((((me.digits - 1) // 10)) * me.parent().par.h / 9) - y
```

Each cell spans to the *next* boundary rather than using a fixed width, so integer pixel rounding
of `size/9` leaves no seams. Verified with a per-pixel coverage map: 80 buttons, zero overlapping
pixels, only the top-right cell uncovered.

`display` is `parent().par.display` for the 80 real buttons and a hard `False` for the 20 that
don't exist on an MK2, so bank switching still works as originally designed.

---

## Gotchas

### 1. TouchDesigner names MIDI channels 1-based
Hardware note 11 → `ch1n12`. Hardware CC 104 → `ch1ctrl105`. This is why the component's
`p = digits - 1` convention lines up correctly, and it is *not* an off-by-one bug.

### 2. CHOP Execute DATs do not keep their CHOP cooking
A CHOP Execute is a **side-effect consumer** — it watches a CHOP but never pulls data from it. A
CHOP whose only consumer is one of them has nothing demanding its output, and TD stops cooking it
as soon as its network isn't being viewed.

> **Symptom:** everything works while you have the network open, and dies the moment you navigate
> elsewhere.

`mk2_keepalive` fixes this. **If you add a CHOP Execute, add the CHOP it watches to its `LEAVES`
list.** Current list:

```
midiin1, mk2_ccremap, mk2_pagefilter, mk2_pageselect,
launchpad_base0..3/null9, button_state, colour_change, par1, select1
```

### 3. `cook(force=True)` does not dirty upstream
It re-cooks *that node* but reuses cached inputs unless they are already dirty. Forcing only the
leaves left `null9` at 33,021 cooks while its own upstream sat at 3,192 — busily recomputing stale
data. **The `LEAVES` list is ordered head-first** (`midiin1` before the leaves) so there is fresh
data to pull through. Keep that order.

### 4. `align` overrides x/y
The bank containers were on `align = 'gridrows'`, which auto-arranges children by network order
and **ignores x/y entirely**. All four are now `align = 'none'`.

> **Symptom:** a layout change appears to do nothing at all. Check this first.

The original also drove positions from an 11 × 11 grid SOP while there are only 100 buttons, so
they filled 11 per row and the 11th of each row was clipped off-screen. That skew is gone.

### 5. Panel state propagates at frame boundaries
`interactMouse()` followed by reading `panel.state.val` in the *same* script gives misleading
results. Test across two separate calls, or hold the button and read on the next call. Several
false "it's broken" conclusions during this port were this artifact.

### 6. Press flashes are transient
Momentary presses decay within a frame or two — faster than a round-trip. To verify a press
visually, hold it, or toggle `display` (which is persistent) and diff the render.

### 7. Diagnosing "is anything actually running?"
Compare `op.totalCooks` across two separate calls against elapsed `absTime.frame`. A node cooking
0 times over ~1000 frames is dormant. Do **not** use `op.cookFrame` — it is a timeline frame and
wraps on loop, so it says nothing about recency.

A throwaway logger is the fastest way to see what the hardware actually sends:

```python
# CHOP Execute on mk2_ccremap, Off to On only
def offToOn(channel, sampleIndex, val, prev):
    op('zz_midilog').appendRow([channel.name, round(val, 2), int(absTime.frame)])
```

---

## Recipes

**Change which buttons select pages** — edit `mk2_pageselect.channames` and
`mk2_pagefilter.delscope` (keep them identical), and the arithmetic in `mk2_pagechange`
(`page = int(channel.name[-2:]) - 91`). Update `PAGE_LEDS` in `chopexec3_`, `chopexec5_` and
`mk2_pageleds` to the new LED numbers.

**Add a fifth page** — add a bank container, add its one-hot constant in `base2`, extend the
`Presets` menu, add a `switch7_export` row for its `layer`, and add its `null9` to
`mk2_keepalive`.

**Build a real mask** — the `mask` container sits on `layer = 2` (above the banks, which use 0/1),
with `enable = Off` and `clickthrough = On`. **Keep those two or the pads stop responding.**
Rewire the input of the `mask` null inside it: transparent = button shows through, opaque =
hidden. Grid geometry to match: 9 × 9, cell = `W/9`, columns left→right are hardware columns 1–9,
rows bottom→top are hardware rows 1–9, top-right cell empty.

**Port back to a Launchpad Pro** — change `2, 24` to `2, 16` in `chopexec3_` and `chopexec5_`,
revert `table2` rows 91–98 to `91`–`98`, drop `mk2_ccremap`, and restore the bottom row and left
column in the button `display` logic.

**Drive buttons from an external source** — the generic way to press a button from outside is
`op('buttonNN').interactMouse(.5, .5, left=True)` then `left=False`. Press-and-release in the same
callback is a 1→0 blip within one frame that nothing downstream will sample; for a momentary
target, release on the source's off-transition instead.
