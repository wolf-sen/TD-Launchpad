# Novation Launchpad MK2 — TouchDesigner Component

`launchpad_mk2.tox` turns a Launchpad MK2 into a set of TouchDesigner buttons: press a pad on the
hardware and the matching on-screen button presses, with the pad's LED following along. Four
independent pages of buttons are built in, switched from the device itself.

Ported from Owen Kirby's 2017 Launchpad **Pro** component. Internals and porting notes are in
[DEVELOPMENT.md](DEVELOPMENT.md).

---

## Setup

1. Connect the Launchpad MK2 **before** starting TouchDesigner.
2. Open the MIDI Device Mapper (`Dialogs → MIDI Device Mapper`) and map the Launchpad to a
   device ID, with both **In** and **Out** enabled. Output is required — without it the pads
   won't light.
3. Drag `launchpad_mk2.tox` into your network.
4. Check that `midiin1` and `midiout1` inside the component point at the ID you mapped.

The pads should light and respond immediately.

---

## The panel

The component draws the MK2's layout as a 9 × 9 grid, filling the panel edge to edge:

```
 ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
 │ ▲ │ ▼ │ ◀ │ ▶ │Ses│Us1│Us2│Mix│   │  ← top row (round buttons)
 ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
 │   │   │   │   │   │   │   │   │ ● │
 │   │   │   │   │   │   │   │   │ ● │
 │   │   │     8 × 8 pads      │   │ ● │  ← right column
 │   │   │   │   │   │   │   │   │ ● │
 └───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

80 buttons: 64 pads, 8 across the top, 8 down the right. The top-right cell is empty because the
MK2 has no button there. The MK2 has no bottom row or left column, so neither is drawn.

The panel resizes cleanly — change the component's Width and Height and the grid rescales.

---

## Parameters

On the component's **PRESETS** page:

| Parameter | What it does |
|---|---|
| **Presets** | Which page (1–4) is active. Also driven by the hardware arrow keys. |
| **Brightness** | Global LED brightness, 0–1. Scales every colour sent to the device. |
| **Last Channel** | Read-only readout: the name of the last output channel to go on, e.g. `1_v47`. Press a pad, read the name, paste it wherever you are routing the output. |

---

## Pages

Four independent pages, each with its own button states and colours. The **four arrow keys**
(Up, Down, Left, Right) select pages 1–4 and behave as a radio group: pressing one selects that
page and lights it brightly while the other three dim.

Switching pages swaps the whole grid — every button's state and colour changes to that page's,
and the hardware LEDs repaint to match. Buttons keep their state per page, so a toggle left on
in page 2 is still on when you come back to it.

The arrow keys are reserved and don't act as normal buttons. **Session, User 1, User 2 and
Mixer are available** as ordinary per-page buttons. They do work, but the MK2 also acts on them
internally to switch its own layout, so prefer pads if you have the choice.

To change how many pages there are, see [DEVELOPMENT.md](DEVELOPMENT.md) — it's more than just
editing the menu.

---

## Button types

Each button is a standard TouchDesigner Button COMP, so its behaviour is set by its own
**Button Type** parameter:

| Type | Behaviour |
|---|---|
| Momentary | on while held, off on release |
| Toggle Down | flips state on each press |
| Radio | only one button in the container can be on |

To change one: go into the bank for the page you want (`launchpad_base0` = page 1, `base1` =
page 2, and so on), select the button, and set **Button Type**.

Buttons are named `buttonN`, where the grid position comes from `N - 1`:
`column = (N-1) % 10`, `row = (N-1) // 10`, counting **row 0 at the bottom**. So `button12` is
the bottom-left pad and `button92` is the top-left arrow.

**Each page has its own copy of every button**, so changing a button's type only affects that
page. Set it in all four banks if you want it consistent.

---

## Colours

Every bank has two 10 × 10 TOPs, `on` and `off`, that hold the colour of each button — one pixel
per button, at the same column/row as the grid. Feed them anything you like; animated TOPs work
and the LEDs follow.

Note the naming is about the button **graphic**, not its state: a button at rest shows the `on`
TOP, and shows the `off` TOP while it is active. That's inherited from the original component
and the LEDs follow the same convention, so the hardware always agrees with the screen.

The Launchpad accepts 0–63 per colour channel; the conversion from TOP values is handled for you,
and **Brightness** scales the result.

---

## Outputs

The component outputs one CHOP with a channel per button per page:

| Channel | Meaning |
|---|---|
| `1_b12` | page 1, button 12 — the button's state |
| `2_b12` | the same button on page 2 |
| `1_v12` | velocity |
| `1_a12` | aftertouch |

The prefix is the page number, so `3_b45` is button 45 on page 3. Use a Select CHOP to pull out
the channels you care about.

Velocity and aftertouch channels exist but the **MK2 is not pressure sensitive** — it sends a
fixed velocity and no aftertouch, so those channels stay static. They're kept for compatibility
with the original Pro component.

---

## Saving

The component holds its own state, so save it back to the tox after making changes:

```python
op('/project1/launchpad_mk2').save('launchpad_mk2.tox')
```

Or right-click the component → **Save Component .tox**.

---

## Known limitations

- No velocity or aftertouch (hardware limitation of the MK2).
- The four arrow keys are reserved for page switching.
- Adding a fifth page needs changes in three places — see [DEVELOPMENT.md](DEVELOPMENT.md).
- `launchpad pro.pdf` in this repo is the **Pro** programmer's reference. The MK2 uses different
  numbers for the top row and a different SysEx device ID; the MK2 guide is the authority.
