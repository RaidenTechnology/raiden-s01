# Raiden S0-1 — Build Journal

**Project:** an open-hardware ESP32-S3 development board, designed from scratch in KiCad 9.
**Author:** Raiden Technology
**Repo:** https://github.com/RaidenTechnology/raiden-s01
**License:** CERN-OHL-P v2

> **A note on this journal, up front:** the design work happened over 11–15 July 2026. I
> published the repository as a single release commit on 15 July and only wrote this journal
> afterwards, from my own working notes. So the entries below are an honest reconstruction of
> real sessions, not a live log — I'd rather say that than pretend the commit history shows
> something it doesn't. Everything technical in here is verifiable against the files in
> `hardware/` and `fab/`.
>
> The same applies to images: I didn't capture screenshots while working, so the only renders
> I have are of the finished board, and they appear in the session where they were actually
> generated. I'd rather show three real images than dress up intermediate ones I never took.
> The full schematic is in [`fab/Raiden S0-1 Schematic.pdf`](fab/Raiden%20S0-1%20Schematic.pdf)
> and every claim below can be checked against the KiCad files in `hardware/`.

**What makes this board not "just another devboard":** it is built around a **bare ESP32-S3FN8
in a QFN-56 package**, not a pre-made module. That means the 0.4 mm pitch fanout, the crystal
network, the USB-C front end, the RF matching network and the 50 Ω antenna feed are all mine to
get right. It is also dual-purpose: standalone, or seated on a breadboard, with edge power rails
so sensors can be fed without a breadboard at all.

---

## Session 1 — 11 July 2026 · Schematic, and the first PCB layout
**Time:** _(fill in)_

Built the schematic around the ESP32-S3FN8 (QFN-56, 8 MB embedded flash).

Design decisions made this session:
- **Regulator:** AZ1117-3.3 in SOT-223, switched away from an AP2112K. Bigger tab, better
  thermals for Wi-Fi bursts.
- **Protection:** 750 mA PTC resettable fuse in series with VBUS; PRTR5V0U2X ESD array shunting
  the USB D+/D− lines.
- **USB-C:** CC pull-downs, B6/A6 and B7/A7 bridged.
- **Reset / boot:** RC on CHIP_PU (10 kΩ + 1 µF) for the reset button, 100 nF debounce on the
  boot button.
- **Dropped the buzzer.** It was in the plan; it earned no space on the board.

**Trap I walked into:** the KiCad symbol names pins 36/37 "SPICLK_N" and "SPICLK_P", which reads
like they belong to the flash interface and are untouchable. They are not — on the FN8 they are
free GPIO48 and GPIO47. Only seven pins are genuinely off-limits (the internal flash: SPICS0/1,
SPICLK, SPIQ, SPID, SPIHD, SPIWP). I nearly threw away two usable I/O.

ERC reached 0. Added the RF chain (1 pF + 2.0 nH + 1 pF into a u.FL connector) with starting
values to be trimmed against a real measurement.

Laid out the PCB: 43 components on a 36 × 60 mm board, every footprint assigned.

**What I learned:** I tried to script the routing. On a 0.4 mm pitch QFN it produced shorts, and
I reverted the whole attempt. Hand routing it is — and the order matters: USB differential pair
first, then the 50 Ω RF feed, then power, then the crystal, then GPIO. The constrained,
irreversible things go first.

I also had both GPIO headers on the wrong sides relative to the chip's pin banks, so every
single signal was crossing the board to reach its header. Swapping them removed the problem
entirely. Cheap fix, but only because I caught it before routing.

---

## Session 2 — 12 July 2026 · Invisible disconnections, and why 2 layers wasn't enough
**Time:** _(fill in)_

Checked the traces I had drawn by hand. Ten of them (GPIO1–8, 14, 17) **stopped 60–100 µm short
of the QFN pad copper.** On screen they looked connected. They were not. Several joints were
also 3–8 µm off-grid, which was quietly generating clearance violations.

**What I learned:** turn on magnetic pads, and always pull traces *outward from the chip* rather
than inward toward it. "It looks connected" is not a verification method — this is the single
most useful habit I picked up on this board.

Then I ran a union-find analysis over all the GND copper to see what was actually joined to
what. The result was unpleasant: **the bottom layer was not a ground plane at all.** It was
signal from top to bottom (3.3 V, EN, crystal, GPIO0, USB), with ground merely filling the
leftover gaps in three separate islands. The SMD capacitors' ground pads couldn't reach it —
there were no vias. I added 7 GND vias, each one geometry-checked (inside the GND polygon,
≥ 0.2 mm from any signal, actually connected to the main island). Disconnected GND nodes went
19 → 12.

I tried to solve the +3.3 V distribution with a copper zone on the front layer. **Zero polygons
filled.** The area around the chip is so dense with header fanout that not even a 0.25 mm pour
fits. That failure was the useful part: it was proof, not opinion, that the board needed more
layers.

**Moved to 4 layers:** inner layer 1 = full ground plane, inner layer 2 = full +3.3 V plane.

**What I learned (the part I had wrong):** going to 4 layers gives you power and ground planes,
but the *signal* layer count is still two. The congestion around the QFN does not improve at
all. I expected the extra layers to relieve the fanout; they didn't. What they fixed was
distribution, not routing.

Two tooling lessons from this session:
- `kicad-cli` **does not refill zones.** It validates against the last saved fill, so any new
  zone connection reads as broken until refilled in the GUI. I chased phantom errors before
  I understood this.
- When editing the board file as text, the zone anchor must be matched as a newline + single
  tab. Matching a bare tab hits the keepout zone *inside a footprint* instead and corrupts the
  file. I did this, and had to revert.

---

## Session 3 — 12–13 July 2026 · Hunting isolated copper, and blind vias
**Time:** _(fill in)_

`kicad-cli` DRC kept reporting the ground zone as having a missing connection, but the
coordinates it gave were useless — it always points at the zone anchor, not the actual fault.

I wrote a proper check instead: union-find across *all* ground copper on all layers, joining
vias, pads and tracks by **shape collision** rather than by centre point. A centre-point test
gives wrong answers, because two shapes can overlap without either centre being inside the
other.

That found the real faults: three genuinely isolated pockets of front-layer ground.

Two of them sat directly over congested bottom-layer copper. Dropping a normal through via there
would have **shorted ground into the EN and 32 kHz crystal traces.** So I used blind vias, front
layer down to the inner ground plane only.

Then I undid that. **Standard 4-layer fabrication cannot do blind vias** — it's a sequential
lamination process and costs multiples of a plain board. I converted both back to through vias
and moved them to points I searched for programmatically: somewhere that a via *and* its
connecting stub would both clear every existing track, via and pad.

**What I learned:** design for the process, not just for the tool. KiCad let me draw something
that no affordable fab would build. That distinction was invisible to me until I went looking
for the price.

Also caught this session: **the regulator's thermal tab pad had no net assigned — it was
floating.** The part would have worked and then cooked. Assigned it to +3.3 V, added a thermal
via and a local zone.

---

## Session 4 — 13 July 2026 · Cleanup and optimisation
**Time:** _(fill in)_

- Added 4 × M2 mounting holes (non-plated), positioned by searching the corners for spots that
  didn't collide with the boot button or the regulator cluster.
- **Silkscreen: 32 violations → 0.** Pulled the header pin numbers inward, shifted the rear I/O
  name column. Four labels had to be deleted outright — the gap between one header pad and the
  regulator cluster is 0.46 mm, and no legible text fits in that. The rear-side I/O names and
  the neighbouring front numbers make up for it.
- **+5 V route: 61 segments → 12**, by string-pulling the path against the real obstacle set.
  It had been an ugly zigzag left over from an automated attempt.
- Added 5 ground stitching vias along the RF feed line.

**What I learned:** my via-placement checker validated against tracks, pads and vias — but it
knew nothing about **keepout zones**, so it happily placed a via inside the u.FL antenna
keepout. The keepout under the antenna is deliberate: it's an intentional gap in the copper, and
it is *not* a mistake to be filled in. I had to exclude the RF region by hand. A checker is only
as good as the constraint list you gave it, and the constraints you forget are invisible.

---

## Session 5 — 14 July 2026 · Reviewing my own board
**Time:** _(fill in)_

Went through the board as if reviewing someone else's work, against a checklist. Scored it
**84/100** and wrote down the deductions honestly rather than defending the design.

Fixed:
- Deleted a dangling 3.3 V trace stub.
- Added title blocks to both schematic and board (name, revision, date, crystal and impedance
  notes).
- Added **manufacturer part numbers to 37 components** — until then the BOM was unbuildable by
  anyone but me.
- Put a visible "CL = 8 pF" warning on the schematic canvas next to the 40 MHz crystal. The
  board's 10 pF load capacitors only give the right oscillator load for an 8 pF crystal; buying
  the wrong one produces a board that looks fine and won't boot.
- Added three fabrication notes to the board comments layer.
- Generated the BOM.

Then I verified that the netlist was **node-for-node identical** to before. Adding metadata
should change nothing electrically, and I wanted that proven rather than assumed.

**Known weaknesses I did not fix, and am not hiding:**
- No Schottky diode between the 5 V rail and USB VBUS — if the board is powered from both at
  once, there's a back-feed path.
- The 100 nF decoupling capacitors sit 4.5–6 mm from the chip's power pins. That's further than
  I want; it's a consequence of the fanout congestion.
- The RF feed is drawn for 50 Ω but **not impedance-verified** against a defined stackup. The
  fab notes ask the manufacturer to adjust trace width to their actual stackup.

These are Rev B work, along with a power LED and tighter decoupling.

---

## Session 6 — 14–15 July 2026 · Fabrication package and release
**Time:** _(fill in)_

Filled all six zones through the pcbnew Python API, because `kicad-cli` has no fill command at
all — this was the last piece of the "why do my zone errors never clear" puzzle from Session 2.

Exported the full production package into `fab/`: 11 Gerber layers, Excellon drill files, a
drill map, pick-and-place data, the schematic as PDF, and the BOM with part numbers. (Small
trap: the drill export takes `--excellon-units`, not `--units`. The generic flag is silently
wrong.)

Rendered three views with `kicad-cli pcb render`. These are the board as it stands today —
what the grant would turn into copper:

**Top** — the QFN-56 in the centre, both 15-pin GPIO headers, USB-C at the edge, the u.FL
antenna connector and its deliberate copper keepout, and the silkscreen after the cleanup pass
in Session 4 (pin numbers on the front, I/O names on the back):

![Top view](images/board-top.png)

**Bottom** — the rear I/O name column, the edge power rails, and the four M2 mounting holes:

![Bottom view](images/board-bottom.png)

**Perspective:**

![Hero render](images/board-hero.png)

(A render trap worth noting: `--rotate` needs its value in nested quotes, e.g.
`--rotate "'-35,0,-30'"`, or the argument parser rejects it.)

Published the repository under CERN-OHL-P v2.

**Final verification state:** ERC 0 errors · DRC 0 errors · 0 unconnected items. Five remaining
DRC notices are intentional local footprint overrides — the u.FL RF keepout and the four
mounting holes — and they are documented as such rather than suppressed.

---

## Status

**Rev A is design-complete and verification-clean, and has never been manufactured.** The
Gerbers in `fab/` have not yet been sent to a fab. That first run is exactly what I'm asking a
build grant to cover.

**Rev B** is in progress in a separate project: the same board with the GPIO headers spaced
25.4 mm apart so it seats on a breadboard and stays pin-compatible with a DevKitC. Two nets
still need routing there.

## What I'd tell someone starting a board like this

1. **Verify, don't look.** Ten of my traces were disconnected by 60 µm and looked perfect.
2. **Automate the checking, hand-route the copper.** Every routing script I wrote on 0.4 mm
   pitch produced shorts. Every *analysis* script I wrote found real faults I'd never have
   spotted by eye.
3. **A tool letting you draw it doesn't mean a factory can build it.** Blind vias taught me that.
4. **When something fails to fit, that's data.** The zone that filled zero polygons is what told
   me the board had to go to four layers.
