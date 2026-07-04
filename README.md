# picorv32a: RTL to GDSII with OpenLANE + Sky130

A record of me taking the `picorv32a` RISC-V core through the open-source RTL-to-GDSII flow — synthesis, floorplanning, placement, and building/dropping in a custom standard cell — as part of the VSD Digital VLSI SoC Design workshop. Everything past the theory section is my own run, on my own VM, with numbers pulled from my own reports.

I'm through Day 4 (custom cell integrated and floorplanned, haven't hit CTS/routing with it yet), so this will keep growing.

---

## Table of Contents

1. [Theory Primer](#theory-primer)
2. [Tools & Environment](#tools--environment)
3. [Day 1 — Setup and First Synthesis](#day-1--setup-and-first-synthesis)
4. [Day 2 — Floorplan, PDN, Placement](#day-2--floorplan-pdn-placement)
5. [Day 3 — Custom Standard Cell Design](#day-3--custom-standard-cell-design)
6. [Day 4 (in progress) — Wiring the Custom Cell into the Flow](#day-4-in-progress--wiring-the-custom-cell-into-the-flow)
7. [Acknowledgements](#acknowledgements)

---

## Theory Primer

Before touching the tools, it helps to know what's actually happening physically and conceptually. Here's the background I needed before any of the lab steps made sense.

### What people call "the chip" isn't the chip

Pop open any board — an Arduino, a phone charger, whatever — and the black rectangle you point to and call "the chip" is really the **package**. It's a protective shell. The actual piece of silicon sits at the center of that package, and it talks to the outside world through **wire bonding** — hair-thin wires connecting pads on the silicon to pins on the package.

Inside that silicon itself:

| Term | What it is |
|---|---|
| **Pad** | The interface ring around the edge of the chip where every external signal enters/exits |
| **Core** | The region inside the pad ring — this is where all the actual logic lives |
| **Die** | Core + pads together — this is the basic manufacturing unit; a wafer gets cut into individual dies |

### Foundries, Foundry IPs, and Macros

A **foundry** is the fab that physically manufactures the silicon. Some blocks (PLLs, ADCs, SRAM compilers) need deep process-specific expertise to design correctly for a given foundry's node — these are called **Foundry IPs**. Reusable purely-digital blocks that don't need that specialized analog know-how are called **Macros**.

### From a C program to a switching transistor

There's a chain of translation between "I wrote a function in C" and "a transistor switched":

1. C code → compiled to an assembly-level ISA (RISC-V in this workshop)
2. Assembly → assembled into raw binary (0s and 1s)
3. That instruction set gets an actual RTL implementation (Verilog/VHDL describing the hardware that executes those instructions)
4. RTL → synthesis → place & route → GDSII (the manufacturable layout)

The OS, compiler, and assembler are the software stack that quietly does steps 1–2 every time you run a program — the CPU only ever sees step 2's output.

This workshop is really three sub-projects stacked on each other:
- The RISC-V ISA itself
- RTL + synthesis of a RISC-V core (`picorv32a`)
- Physical design of that synthesized netlist, down to GDSII

### Why an open-source PDK is a bigger deal than it sounds

A **PDK (Process Design Kit)** is the contract between a chip designer and a fab — device models, design rules, cell libraries, everything needed to guarantee a design will actually manufacture correctly on that fab's process. For decades PDKs were locked behind NDAs, which meant nobody could learn real physical design without an employer or university sponsoring fab access.

That changed in mid-2020, when Google and SkyWater released **Sky130** — a fully open 130 nm PDK, free to download and use. Combined with open synthesis/PnR tools (Yosys, OpenROAD, Magic, etc.) and open RTL, this is what makes a genuinely open-source ASIC flow possible for the first time. That's the flow this repo runs.

### The RTL-to-GDSII flow at a glance

| Stage | What happens | Tool used here |
|---|---|---|
| Synthesis | RTL → gate-level netlist, mapped to standard cells | Yosys / ABC |
| Floorplanning | Die/core size decided, I/O pins placed, power planning | OpenROAD |
| Placement | Every standard cell gets an (x, y) — global, then legalized | OpenROAD |
| Clock Tree Synthesis (CTS) | Balanced buffer tree built to distribute the clock with minimal skew | TritonCTS |
| Routing | Global routing (regions/paths) then detailed routing (exact wires/vias) | FastRoute → TritonRoute |
| Parasitic extraction | Pull real R/C off the routed wires | SPEF extractor / OpenRCX |
| Sign-off | Confirm the layout is actually manufacturable and correct | Magic (DRC), Netgen (LVS), OpenSTA (timing) |

Two floorplanning numbers worth knowing before Day 2:

- **Utilization factor** = area occupied by the netlist ÷ total core area. Too high and there's no room for buffering/routing; typical target is roughly 0.5–0.6.
- **Aspect ratio** = core height ÷ core width. 1.0 = square core.

### Sign-off checks, briefly

- **DRC (Design Rule Check)** — does the layout obey the foundry's minimum spacing/width rules?
- **LVS (Layout vs. Schematic)** — does the physical layout actually implement the same connectivity as the netlist it came from?
- **STA (Static Timing Analysis)** — does every path in the design meet setup and hold timing at the target clock?

---

## Tools & Environment

| Tool | Role |
|---|---|
| OpenLANE v0.21 (Docker) | Orchestrates the whole RTL2GDS flow |
| Sky130A PDK | Open-source 130 nm process design kit |
| Yosys | RTL synthesis |
| OpenROAD | Floorplan, placement, CTS |
| OpenSTA | Static timing analysis |
| Magic | Layout viewing, DRC, LVS, extraction |
| ngspice | SPICE-level circuit simulation |
| TritonRoute | Detailed routing |

Run on a VirtualBox VM (`vsdworkshop1`) with the OpenLANE working directory at `Desktop/work/tools/openlane_working_dir/openlane`.

---

## Day 1 — Setup and First Synthesis

### Launching OpenLANE

```tcl
cd Desktop/work/tools/openlane_working_dir/openlane
docker
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a
```

`prep` merges the standard cell LEF with the fill/decap/diode macro LEFs, points OpenLANE at the Sky130A PDK, and creates a fresh timestamped run folder.

**[fig 01]** — terminal right after `flow.tcl -interactive`, OpenLANE ASCII banner still loading, LEF-merge log scrolling (SITE/MACRO match counts for the standard cell, fill, decap, and diode LEFs).
*→ place directly under "Launching OpenLANE"*

**[fig 02]** — same terminal moments later: `mergeLef.py: Merging LEFs complete`, `Preparation complete`, prompt now sitting at `% run_synthesis`.
*→ place right after the code block above*

### Running synthesis

```tcl
run_synthesis
```

**[fig 03]** — OpenSTA's post-synthesis summary: `tns -759.46`, `wns -24.89`, `[INFO]: Synthesis was successful`.
*→ place after `run_synthesis`*

### Reading the reports

```bash
cd runs
ls designs
cd designs/picorv32a
ls runs
cd 01-07-06-06
pwd
ls -ltr
cd reports/synthesis
ls
cat 1-yosys_4.stat.rpt
```

**[fig 04]** — navigating `runs → designs/picorv32a → runs`, listing all the sample designs OpenLANE ships with.
*→ place after the navigation block, before showing report contents*

**[fig 05]** — `ls -ltr` inside the run folder (`results/`, `reports/`, `logs/`, `tmp/`, `config.tcl`), then `cat 1-yosys_4.stat.rpt` starting to print: `Number of wires: 14596`, `Number of cells: 14876`, and the cell breakdown beginning.
*→ place right after*

**[fig 06]** — stat report continuing (mux/nand/nor/o21a family counts scrolling by).
*→ place directly under fig 05*

**[fig 07]** — end of the stat report, finishing at `or4bb_2`, and the line: `Chip area for module 'picorv32a': 147712.918400`.
*→ place last in this section*

### Synthesis summary table

| Metric | Value |
|---|---|
| Total cell count | 14,876 |
| D flip-flops (`sky130_fd_sc_hd__dfxtp_2`) | 1,613 |
| **Flop ratio** = DFFs ÷ total cells | 1613 / 14876 = **0.1084** |
| **% DFFs** | **10.84%** |
| Synthesized cell area (Yosys) | 147,712.92 µm² |
| Clock period used | 5.0 ns (see Day 2 — this gets fixed) |
| WNS / TNS at this clock | −24.89 ns / −759.46 ns |

That heavily negative WNS is expected — a 5 ns clock is unrealistic for this design at this stage, which is exactly why Day 2 starts by fixing it.

---

## Day 2 — Floorplan, PDN, Placement

### Fixing the clock period

**[fig 08]** — `config.tcl` open, `CLOCK_PERIOD "5.000"` visible.
*→ place at the top of this section*

**[fig 09]** — `sky130A_sky130_fd_sc_hd_config.tcl` open, showing `CLOCK_PERIOD "24.73"` — this is a value OpenLANE derives internally, and it silently overrides the top-level config unless you patch this file too.
*→ place right after fig 08 — this is the file that trips most people up*

```tcl
run_synthesis
```

**[fig 10]** — OpenSTA re-run against the corrected 11.0 ns delay budget (period × IO_PCT), this time closing at `tns 0.00` / `wns 0.00` — clean timing.
*→ place after*

### Floorplan

```tcl
run_floorplan
```

**[fig 11]** — floorplan log: PDN vsrc-placement warnings (`is not located on a power stripe, moving to closest stripe...`), finishing with `PDN generation was successful`.
*→ place after `run_floorplan`*

**[fig 12]** — navigating into `results/floorplan` and listing (`merged_unpadded.lef`, `picorv32a.floorplan.def`, `picorv32a.floorplan.def.png`).
*→ place before the DEF contents*

**[fig 13]** — `picorv32a.floorplan.def` opened, `DIEAREA ( 0 0 ) ( 666685 671405 )` and the `ROW` definitions visible.
*→ place right after fig 12*

### Die area — from my own DEF

| Quantity | Value |
|---|---|
| Unit distance | 1000 DB units = 1 µm |
| Die width | 666685 ÷ 1000 = 666.685 µm |
| Die height | 671405 ÷ 1000 = 671.405 µm |
| **Die area** | 666.685 × 671.405 ≈ **447,615 µm²** |

### Viewing the floorplan in Magic

```tcl
magic -T .../sky130A.tech lef read ../../tmp/merged.lef def read picorv32a.floorplan.def &
```

**[fig 14]** — Magic `layout1` with just the die outline + rows loaded, tkcon logging label placements alongside.
*→ place after the magic command*

**[fig 15]** — zoomed-in view: decap cells and tap cells (`sky130_fd_sc_hd__tapvpwrvgnd_1`) placed at equal spacing along the rows.
*→ place right after fig 14*

### Placement

```tcl
run_placement
```

**[fig 16]** — placement run — same PDN stripe warnings replaying (normal, not a new failure), ending at `% run_placement`.
*→ place after*

**[fig 17]** — the full placement stats block.
*→ place directly after fig 16*

### Placement summary table (from fig 17)

| Metric | Value |
|---|---|
| Fixed instances | 6,354 |
| Nets | 15,450 |
| Design area | 420,473.3 µm² |
| Fixed area | 9,141.3 µm² |
| Movable area | 147,856.8 µm² |
| **Utilization** | 36% |
| **Utilization, padded** | 55% |
| Rows | 238 |
| Row height | 2.7 µm |
| HPWL before legalization | 778,498.6 µm |
| HPWL after legalization | 765,608.5 µm |
| **HPWL improvement** | −1.7% |

### Loading placement in Magic

```tcl
magic -T .../sky130A.tech lef read merged_unpadded.lef def read picorv32a.placement.def &
```

**[fig 18]** — placement DEF loading, tkcon confirming `Processed 21700 subcell instances`, `17241 nets`, `DEF read: Processed 41860 lines`.
*→ place after the magic command*

**[fig 19]** — two views: full-chip zoomed out (dense, every cell placed) and zoomed in with individual instances labeled (`o2a_1`, `a2bb2o_1`, `ckbuf_4`, `and4_2`), legally abutting.
*→ place last, closes out Day 2*

---

## Day 3 — Custom Standard Cell Design

The goal here was building my own inverter cell in Magic, extracting it to a SPICE netlist, and confirming with ngspice that it actually behaves like an inverter before it goes anywhere near the flow.

### Cloning the layout

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign
cd vsdstdcelldesign
cp <pdk_path>/libs.tech/magic/sky130A.tech .
magic -T sky130A.tech sky130_inv.mag &
```

**[fig 20]** — terminal running clone + copy (a couple of path typos first before it goes through), next to the Magic layout showing NMOS/PMOS regions and the `A` (input), `Y` (output), `VPWR`, `VGND` labels.
*→ place after the clone block*

### Extracting to SPICE

Inside tkcon:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

**[fig 21]** — tkcon logging the extraction (`extspice finished`) beside the layout view, plus the zoomed-out "topmost cell in window" shot from over-zooming after extraction.
*→ place after the extract commands*

### The SPICE netlist and simulation

**[fig 22]** — the generated `sky130_inv.spice` netlist: two MOSFETs (`M1000`/`M1001`, `pshort_model`/`nshort_model`), `VDD`/`VSS` supplies, a `PULSE` stimulus on node `a`, load/parasitic capacitors, `.tran 1n 20n`, and the `.control / run / plot v(a) v(y) / .endc` block. Below it, ngspice's initial DC solution and `No. of Data Rows: 160`.
*→ place under this heading*

```bash
ngspice sky130_inv.spice
plot y vs time a
```

**[fig 23]** — transient data table (`time`, `v(a)`, `v(y)`) and the waveform plot: red trace = input pulse on `a`, blue trace = inverted output on `y`, switching every cycle.
*→ place after the plot command — this is the proof the cell inverts correctly*

### How to pull the standard timing numbers off this waveform

| Metric | Definition |
|---|---|
| Rise transition time | time(80% Vdd rising) − time(20% Vdd rising) |
| Fall transition time | time(20% Vdd falling) − time(80% Vdd falling) |
| Rise cell delay | time(output 50%) − time(input falling 50%) |
| Fall cell delay | time(output 50%) − time(input rising 50%) |

I'll fill in the actual measured numbers once I've re-run this with cursors on the ngspice plot rather than eyeballing the data table — didn't want to guess values here.

---

## Day 4 (in progress) — Wiring the Custom Cell into the Flow

### Checking track alignment

Before OpenLANE will accept a custom cell into placement, its ports need to land on the routing track grid, and its dimensions need to be clean multiples of the track pitch.

```tcl
grid 0.46um 0.34um 0.23um 0.17um
```

**[fig 24]** — tkcon `help grid` output, grid applied, Magic view of the inverter with the track grid now overlaid confirming port alignment.
*→ place after the grid command*

| Condition | Requirement | This cell's values | Status |
|---|---|---|---|
| 1 | Ports sit on horizontal/vertical track intersections | confirmed visually against the grid overlay | ✅ |
| 2 | Cell width = odd multiple of horizontal track pitch | pitch = 0.46 µm, width = 1.38 µm = 0.46 × 3 | ✅ |
| 3 | Cell height = even multiple of vertical track pitch | pitch = 0.34 µm, height = 2.72 µm = 0.34 × 8 | ✅ |

### Writing and inspecting the LEF

```tcl
save sky130_vsdinv.mag
lef write
```

```bash
ls
cat sky130_inv.lef
```

**[fig 25]** — directory listing (`.spice`, `.tech`, `.ext`, `.lef`, `.mag` all present), `cat sky130_inv.lef` starting: `MACRO sky130_inv`, `PIN A` / `PIN Y` with `RECT` geometry on `li1`.
*→ place after `lef write`*

**[fig 26]** — rest of the LEF: `PIN VPWR` / `PIN VGND` across `nwell`, `li1`, `mcon`, `met1`, closing `END sky130_inv` / `END LIBRARY`.
*→ place directly after fig 25*

### Wiring it into `picorv32a`

```bash
cp sky130_vsdinv.lef ~/.../designs/picorv32a/src/
cp libs/sky130_fd_sc_hd__* ~/.../designs/picorv32a/src/
```

```tcl
set ::env(LIB_SYNTH)   ".../src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST) ".../src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST) ".../src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL) ".../src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)  [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

**[fig 27]** — terminal copying the LEF/lib files, config.tcl open with the `LIB_*` and `EXTRA_LEFS` lines added.
*→ place after the copy/config block*

### Re-running synthesis with the custom cell in the library

```tcl
prep -design picorv32a -tag 01-07-07-50 -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
run_synthesis
```

**[fig 28]** — `prep` picking up `sky130_inv.lef` as an extra LEF, full synthesis completing at a new area figure.
*→ place after `run_synthesis`*

### Area impact of the custom cell

| Run | Chip area |
|---|---|
| Day 1 — stock library only | 147,712.92 µm² |
| Day 4 — with custom inverter added | 181,729.29 µm² |
| **Change** | **+23%** |

Not surprising — a hand-drawn cell isn't going to be as area-efficient as a foundry-optimized library cell on the first pass, and once it's part of the library, placement/floorplan has to treat it like any other legitimate macro.

### Floorplan with the custom cell

```tcl
init_floorplan
place_io
tap_decap_or
```

**[fig 29]** — `init_floorplan` (die/core area extracted, DEF written), `place_io` (I/O pins placed), `tap_decap_or` (endcaps + tapcells inserted — 528 endcaps, 7872 tapcells this run), and the Magic view of the resulting floorplan.
*→ place after the floorplan commands — last figure for now*



---

## Acknowledgements

This log follows the VSD "Digital VLSI SoC Design and Planning" workshop by Kunal Ghosh (VSD Corp.), with the Day 3–4 custom standard cell lab built around Nickson P. Jose's `vsdstdcelldesign` reference design.
