# Automatika2 — Project Context

## What This Project Is

Interactive PLC (Programmable Logic Controller) ladder logic tools. Two main artifacts:

1. **SwitchLight_Program.L5X** — A Rockwell Automation Studio 5000 (RSLogix 5000) program export in L5X XML format.
2. **SwitchLight_Ladder.html** — A standalone, browser-based interactive ladder logic simulator that mirrors the L5X program with real-time PLC scan simulation.

There is also a generic L5X-to-HTML renderer (`l5x_to_ladder_renderer.py`) that can parse arbitrary L5X files and produce static graphical ladder diagrams.

---

## L5X Program: SwitchLight_Program.L5X

Target: CompactLogix 1769-L33ER, Studio 5000 v33.

### Tags

| Tag | Type | Description |
|-----|------|-------------|
| Switch1 | BOOL | Input selector switch 1 |
| Switch2 | BOOL | Input selector switch 2 |
| Switch3 | BOOL | Input selector switch 3 |
| Strobe1 | BOOL | Strobe enable selector |
| Light1 | BOOL | Output, Green light |
| Light2 | BOOL | Output, Yellow light |
| Light3 | BOOL | Output, Red light |
| FlashBit | BOOL | Internal oscillator state |
| FlashOn | TIMER | ON-phase timer (1000ms PRE) |
| FlashOff | TIMER | OFF-phase timer (1000ms PRE) |

### Rung Logic (8 rungs)

```
Rung 0: XIC(Strobe1) XIO(FlashBit) TON(FlashOn,1000,0)
        — Flash oscillator ON-phase timer (1s)

Rung 1: XIC(FlashOn.DN) OTL(FlashBit)
        — Latch FlashBit when ON-phase done

Rung 2: XIC(Strobe1) XIC(FlashBit) TON(FlashOff,1000,0)
        — Flash oscillator OFF-phase timer (1s)

Rung 3: XIC(FlashOff.DN) OTU(FlashBit)
        — Unlatch FlashBit when OFF-phase done

Rung 4: XIO(Strobe1) [OTU(FlashBit), RES(FlashOn), RES(FlashOff)]
        — Cleanup: reset oscillator when Strobe1 is OFF (parallel branch)

Rung 5: XIC(Switch1) [XIO(Strobe1), XIC(FlashBit)] OTE(Light1)
        — Switch1 -> Light1 (Green): solid or strobe

Rung 6: XIC(Switch2) [XIO(Strobe1), XIC(FlashBit)] OTE(Light2)
        — Switch2 -> Light2 (Yellow): solid or strobe

Rung 7: XIC(Switch3) [XIO(Strobe1), XIC(FlashBit)] OTE(Light3)
        — Switch3 -> Light3 (Red): solid or strobe
```

### Design Pattern: Two-Timer Oscillator

The strobe uses a classic two-timer 50/50 duty cycle pattern:
- FlashOn TON runs while Strobe1 is ON and FlashBit is OFF. When it reaches DN, FlashBit is latched ON.
- FlashOff TON runs while Strobe1 is ON and FlashBit is ON. When it reaches DN, FlashBit is unlatched OFF.
- This creates a 1s ON / 1s OFF repeating cycle.
- When Strobe1 is turned OFF, Rung 4 immediately clears FlashBit and resets both timers.

### Light Output Logic

Each light rung uses a parallel branch: `[XIO(Strobe1), XIC(FlashBit)]`. This means:
- If Strobe1 is OFF, the XIO branch passes power (solid ON).
- If Strobe1 is ON, power flows only when FlashBit is ON (flashing).
- The switch (XIC) gates the entire rung, so the light is always OFF if its switch is OFF.

---

## Interactive Simulator: SwitchLight_Ladder.html

Single-file HTML/SVG/JS application (~500 lines). No external dependencies.

### Architecture

- **PLC scan loop**: `setInterval` at 50ms (SCAN_MS = 50). Each scan evaluates all rung logic sequentially, updates timer accumulators, computes light outputs, then re-renders the full SVG.
- **Full SVG rebuild each scan**: `body.innerHTML = ''` every 50ms. This is intentional for simplicity but has implications for event handling (see below).
- **Tag state**: Plain JS object `tags = { Switch1, Switch2, Switch3, Strobe1, Light1, Light2, Light3, FlashBit }`, all BOOL.
- **Timers**: `timers = { FlashOn: {ACC, PRE:1000, EN, TT, DN}, FlashOff: {...} }`.

### Critical Fix: pointerdown for Toggle

Because the SVG is fully rebuilt every 50ms, standard `click` and `dblclick` events do not work. The element is destroyed between `mousedown` and `mouseup`, so the click event never fires on the original element.

Solution: use `pointerdown` (fires instantly on press, before the element can be destroyed) with a global `clickTracker` object that tracks timestamps per tag. Two `pointerdown` events within 500ms on the same tag = toggle.

```javascript
const clickTracker = {};
function addToggle(g, tag, cy) {
  const r = el('rect', { x: 0, y: cy-22, width: CW, height: 44,
                         fill: 'transparent', cursor: 'pointer' });
  g.appendChild(r);
  g.addEventListener('pointerdown', (e) => {
    e.preventDefault();
    e.stopPropagation();
    const now = Date.now();
    if (clickTracker[tag] && (now - clickTracker[tag]) < 500) {
      tags[tag] = !tags[tag];
      clickTracker[tag] = 0;
    } else {
      clickTracker[tag] = now;
    }
  });
}
```

If you change the rendering approach (e.g., incremental DOM updates instead of full rebuild), you could switch back to standard click/dblclick events.

### Light Colors

```javascript
const LIGHT_COLORS = {
  Light1: { on:'#44dd44', glow:'#44dd44', stroke:'#009900', label:'#008800', off:'#e8f4e8', offStroke:'#2d7d2d' },  // GREEN
  Light2: { on:'#ffee44', glow:'#ffee44', stroke:'#ccaa00', label:'#aa8800', off:'#f4f4e8', offStroke:'#7d7d2d' },  // YELLOW
  Light3: { on:'#ff4444', glow:'#ff4444', stroke:'#cc0000', label:'#cc0000', off:'#f4e8e8', offStroke:'#7d2d2d' },  // RED
};
```

### Spacing Values

These were tuned to avoid visual collisions in branch rungs:
- `cy = 58` (vertical center for single-element rungs)
- Timer rungs: `H = 110`
- Simple rungs: `H = 100`
- Cleanup rung branch spacing: `brSp = 56`
- Light rung branch spacing: `brSp = 60`

### SVG Symbol Drawers

Functions that draw individual ladder instruction symbols:
- `drawXIC(g, tag, x, cy, on)` — Normally open contact
- `drawXIO(g, tag, x, cy, on)` — Normally closed contact
- `drawOTE(g, tag, x, cy, on, color)` — Output energize coil (supports color)
- `drawOTL(g, tag, x, cy, on)` — Output latch coil
- `drawOTU(g, tag, x, cy, on)` — Output unlatch coil
- `drawTON(g, tag, x, cy, on, timer)` — Timer on-delay block with accumulator bar
- `drawRES(g, tag, x, cy, on)` — Timer reset block

### Power Flow Visualization

- Green wires (`--wire-on: #00cc44`) when the rung path is energized
- Gray wires (`--wire-off: #555`) when not energized
- Left rail always green, right rail always present
- Each rung computes its own power flow state based on tag/timer conditions

---

## Generic L5X Renderer: l5x_to_ladder_renderer.py

Python script that parses any L5X file and produces a static (non-interactive) HTML ladder diagram.

- Parses L5X XML, extracts routines, rungs, and RLL instruction text
- Tokenizes RLL text into instruction sequences with branch support
- Renders SVG ladder graphics with proper contact/coil symbols
- Handles parallel branches with `[branch1, branch2]` notation
- Outputs single-file HTML with embedded CSS and SVG

Usage: `python l5x_to_ladder_renderer.py input.L5X output.html`

---

## L5X File Format Notes

- L5X is Rockwell's XML export format for Studio 5000 / RSLogix 5000 programs
- RLL (Relay Ladder Logic) instruction text uses semicolons as rung delimiters
- Branch notation: square brackets with comma-separated parallel paths
- Key instructions: XIC (examine if closed), XIO (examine if open), OTE (output energize), OTL (output latch), OTU (output unlatch), TON (timer on-delay), RES (timer reset)
- Timer tags have sub-elements: .ACC (accumulator), .PRE (preset), .DN (done bit), .TT (timer timing), .EN (enabled)
- L5X files must include DataType definitions for TIMER, tag declarations, and program/routine/rung structure
- Target controller type and revision matter for import compatibility

---

## Known Issues and Gotchas

1. **SVG rebuild kills DOM events** — The 50ms full-rebuild approach means any event handler attached to SVG elements only lives ~50ms. Use `pointerdown` (fires on press) not `click` (fires on release). This is the single most important thing to remember if modifying the interactive simulator.

2. **Timer accumulation uses scan time** — Timers accumulate by `SCAN_MS` (50ms) per scan, not real wall-clock time. This means timer precision is limited to 50ms increments. A 1000ms preset takes exactly 20 scans.

3. **Color mapping is user-specified** — Light1=Green, Light2=Yellow, Light3=Red. This was explicitly corrected during development (an earlier version had Light1=Red, Light3=Green). Do not change without asking.

4. **L5X validation** — The L5X output targets a specific controller (1769-L33ER) and revision (33.01). Importing to a different controller type or older firmware version may require adjustments to the XML header.

5. **Branch rendering spacing** — Parallel branches need extra vertical space. The `brSp` values (56-60px) were tuned by visual inspection. Adding more branches or taller instructions may require re-tuning.
