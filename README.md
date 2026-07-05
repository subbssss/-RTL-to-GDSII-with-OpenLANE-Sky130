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
6. [Day 4 — Custom Cell Integration, STA, ECO Fixes, and CTS](#day-4--custom-cell-integration-sta-eco-fixes-and-cts)
7. [Day 5 — PDN, Routing, and Post-Route Sign-off](#day-5--pdn-routing-and-post-route-sign-off)
8. [Acknowledgements](#acknowledgements)

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

<img width="975" height="719" alt="image" src="https://github.com/user-attachments/assets/ea88a53a-276a-4ca3-a951-3c2da59849e5" />

**[fig 01]** — terminal right after `flow.tcl -interactive`, OpenLANE ASCII banner still loading, LEF-merge log scrolling (SITE/MACRO match counts for the standard cell, fill, decap, and diode LEFs).

<img width="975" height="710" alt="image" src="https://github.com/user-attachments/assets/755a9349-2a77-4789-b110-1df19fd2afd0" />

**[fig 02]** — same terminal moments later: `mergeLef.py: Merging LEFs complete`, `Preparation complete`, prompt now sitting at `% run_synthesis`.

### Running synthesis

```tcl
run_synthesis
```

<img width="975" height="708" alt="image" src="https://github.com/user-attachments/assets/31908d86-072c-4529-8ef1-8636b029b799" />

**[fig 03]** — OpenSTA's post-synthesis summary: `tns -759.46`, `wns -24.89`, `[INFO]: Synthesis was successful`.

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

<img width="975" height="631" alt="image" src="https://github.com/user-attachments/assets/6d21b9b8-2f79-4de4-babd-36b8df253ca1" />

**[fig 04]** — navigating `runs → designs/picorv32a → runs`, listing all the sample designs OpenLANE ships with.

<img width="975" height="702" alt="image" src="https://github.com/user-attachments/assets/e0c49526-14a8-4dda-b08f-d6561201d8fb" />

**[fig 05]** — `ls -ltr` inside the run folder (`results/`, `reports/`, `logs/`, `tmp/`, `config.tcl`), then `cat 1-yosys_4.stat.rpt` starting to print: `Number of wires: 14596`, `Number of cells: 14596`, and the cell breakdown beginning.

<img width="975" height="708" alt="image" src="https://github.com/user-attachments/assets/37f6965f-7f51-4d9f-840b-0d87ddaebf08" />

**[fig 06]** — stat report continuing (mux/nand/nor/o21a family counts scrolling by).

<img width="975" height="708" alt="image" src="https://github.com/user-attachments/assets/23aa3331-cb17-4d80-96b4-96b5f3d9ad9c" />

**[fig 07]** — end of the stat report, finishing at `or4bb_2`, and the line: `Chip area for module 'picorv32a': 147712.918400`.

### Synthesis summary table

| Metric | Value |
|---|---|
| Total cell count | 14,596 |
| D flip-flops (`sky130_fd_sc_hd__dfxtp_2`) | 1,613 |
| **Flop ratio** = DFFs ÷ total cells | 1613 / 14876 = **0.1105** |
| **% DFFs** | **11.05%** |
| Synthesized cell area (Yosys) | 147,712.92 µm² |
| Clock period used | 5.0 ns (see Day 2 — this gets fixed) |
| WNS / TNS at this clock | −24.89 ns / −759.46 ns |

That heavily negative WNS is expected — a 5 ns clock is unrealistic for this design at this stage, which is exactly why Day 2 starts by fixing it.

---

## Day 2 — Floorplan, PDN, Placement

### Fixing the clock period

<img width="975" height="706" alt="image" src="https://github.com/user-attachments/assets/c43abd70-305d-463e-a86b-f7dbb2c0ce45" />

**[fig 08]** — `config.tcl` open, `CLOCK_PERIOD "5.000"` visible. Change to "55.00"

<img width="975" height="710" alt="image" src="https://github.com/user-attachments/assets/49d84417-fe7e-477f-9653-d6af0f352e48" />

**[fig 09]** — `sky130A_sky130_fd_sc_hd_config.tcl` open, showing `CLOCK_PERIOD "24.73"` — this is a value OpenLANE derives internally, and it silently overrides the top-level config unless you patch this file too. Change to "55.00"

```tcl
run_synthesis
```

<img width="975" height="650" alt="image" src="https://github.com/user-attachments/assets/b0016942-f350-4821-b138-299ce2604e58" />

**[fig 10]** — OpenSTA re-run against the corrected 11.0 ns delay budget (period × IO_PCT), this time closing at `tns 0.00` / `wns 0.00` — clean timing.


### Floorplan

```tcl
run_floorplan
```

<img width="975" height="710" alt="image" src="https://github.com/user-attachments/assets/7084d6a8-0ec6-40f4-9a65-9b86d5c5e1ac" />

**[fig 11]** — floorplan log: PDN vsrc-placement warnings (`is not located on a power stripe, moving to closest stripe...`), finishing with `PDN generation was successful`.

<img width="975" height="708" alt="image" src="https://github.com/user-attachments/assets/b1425f00-46b3-458b-a9e3-39df0e221855" />

**[fig 12]** — navigating into `results/floorplan` and listing (`merged_unpadded.lef`, `picorv32a.floorplan.def`, `picorv32a.floorplan.def.png`).

<img width="975" height="708" alt="image" src="https://github.com/user-attachments/assets/0576f52b-5f77-45fb-9867-da3563009e30" />

**[fig 13]** — `picorv32a.floorplan.def` opened, `DIEAREA ( 0 0 ) ( 666685 671405 )` and the `ROW` definitions visible.


### Die area — from DEF

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

<img width="975" height="656" alt="image" src="https://github.com/user-attachments/assets/37b20c08-bcff-4b18-9bdb-cb54059bc97a" />

**[fig 14]** — Magic `layout1` with just the die outline + rows loaded, tkcon logging label placements alongside.

<img width="975" height="654" alt="image" src="https://github.com/user-attachments/assets/70fc0622-15c6-4aa3-979f-b5cd81a6cbe7" />

**[fig 15]** — zoomed-in view: decap cells and tap cells (`sky130_fd_sc_hd__tapvpwrvgnd_1`) placed at equal spacing along the rows.


### Placement

```tcl
run_placement
```

<img width="975" height="650" alt="image" src="https://github.com/user-attachments/assets/c45e5851-942d-436a-8cfa-57b79eb1f2ce" />

**[fig 16]** — placement run — same PDN stripe warnings replaying (normal, not a new failure), ending at `% run_placement`.

<img width="975" height="658" alt="image" src="https://github.com/user-attachments/assets/9506cbc8-293f-4151-af2e-f9d7fbb05b8e" />

**[fig 17]** — the full placement stats block.


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
<img width="975" height="658" alt="image" src="https://github.com/user-attachments/assets/cb7752ff-a059-4368-99b9-03d633b6d099" />

**[fig 18]** — placement DEF loading, tkcon confirming `Processed 21700 subcell instances`, `17241 nets`, `DEF read: Processed 41860 lines`.

<img width="975" height="660" alt="image" src="https://github.com/user-attachments/assets/600d8295-1d52-4bac-852c-dbbbe9e50744" />

<img width="975" height="660" alt="image" src="https://github.com/user-attachments/assets/c8828f4d-1744-4b19-a55a-b4fbfe56da62" />

**[fig 19]** — two views: full-chip zoomed out (dense, every cell placed) and zoomed in with individual instances labeled (`o2a_1`, `a2bb2o_1`, `ckbuf_4`, `and4_2`), legally abutting.


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

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/7ed26570-0eed-46d0-a458-ceede4955024" />

**[fig 20]** — terminal running clone + copy (a couple of path typos first before it goes through), next to the Magic layout showing NMOS/PMOS regions and the `A` (input), `Y` (output), `VPWR`, `VGND` labels.


### Extracting to SPICE

Inside tkcon:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

<img width="975" height="485" alt="image" src="https://github.com/user-attachments/assets/5ae99738-ad99-4352-88f0-2ac79322da6a" />

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/7e5defdb-ab91-4909-a269-e7c53c2c23a3" />

**[fig 21]** — tkcon logging the extraction (`extspice finished`) beside the layout view, plus the zoomed-out "topmost cell in window" shot from over-zooming after extraction.


### The SPICE netlist and simulation

<img width="975" height="671" alt="image" src="https://github.com/user-attachments/assets/7ea1cbb2-5002-4677-b0ad-43b67757ac77" />

<img width="975" height="552" alt="image" src="https://github.com/user-attachments/assets/b2bc11cf-454b-4a51-ac34-44ab13fdb800" />

**[fig 22]** — the generated `sky130_inv.spice` netlist: two MOSFETs (`M1000`/`M1001`, `pshort_model`/`nshort_model`), `VDD`/`VSS` supplies, a `PULSE` stimulus on node `a`, load/parasitic capacitors, `.tran 1n 20n`, and the `.control / run / plot v(a) v(y) / .endc` block. Below it, ngspice's initial DC solution and `No. of Data Rows: 160`.


```bash
ngspice sky130_inv.spice
plot y vs time a
```

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/71e64c1f-2063-4561-b9af-441e130e3fa8" />

<img width="975" height="517" alt="image" src="https://github.com/user-attachments/assets/3d0663cb-03f5-47bd-897e-d70dca4fb646" />

**[fig 23]** — transient data table (`time`, `v(a)`, `v(y)`) and the waveform plot: red trace = input pulse on `a`, blue trace = inverted output on `y`, switching every cycle.


### How to pull the standard timing numbers off this waveform

| Metric | Definition |
|---|---|
| Rise transition time | time(80% Vdd rising) − time(20% Vdd rising) |
| Fall transition time | time(20% Vdd falling) − time(80% Vdd falling) |
| Rise cell delay | time(output 50%) − time(input falling 50%) |
| Fall cell delay | time(output 50%) − time(input rising 50%) |


---

## Day 4 — Custom Cell Integration, STA, ECO Fixes, and CTS

### Checking track alignment

Before OpenLANE will accept a custom cell into placement, its ports need to land on the routing track grid, and its dimensions need to be clean multiples of the track pitch.

```tcl
grid 0.46um 0.34um 0.23um 0.17um
```

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/e63526c9-dfc0-4db4-82c8-f23d4b430ddf" />

**[fig 24]** — tkcon `help grid` output, grid applied, Magic view of the inverter with the track grid now overlaid confirming port alignment.


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

<img width="975" height="504" alt="image" src="https://github.com/user-attachments/assets/3064a968-bdb6-4ef1-8cf4-5981ac436647" />

**[fig 25]** — directory listing (`.spice`, `.tech`, `.ext`, `.lef`, `.mag` all present), `cat sky130_inv.lef` starting: `MACRO sky130_inv`, `PIN A` / `PIN Y` with `RECT` geometry on `li1`.

<img width="975" height="504" alt="image" src="https://github.com/user-attachments/assets/5f360d32-f4a0-4c81-a6e5-9e65046c5d40" />

**[fig 26]** — rest of the LEF: `PIN VPWR` / `PIN VGND` across `nwell`, `li1`, `mcon`, `met1`, closing `END sky130_inv` / `END LIBRARY`.

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

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/dff50877-a7ac-4b5a-aa09-bf0960a040bb" />

**[fig 27]** — terminal copying the LEF/lib files, config.tcl open with the `LIB_*` and `EXTRA_LEFS` lines added.


### Re-running synthesis with the custom cell in the library

```tcl
prep -design picorv32a -tag 01-07-07-50 -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
run_synthesis
```

<img width="975" height="506" alt="image" src="https://github.com/user-attachments/assets/51fa89dd-ae3d-4c99-b132-facd85231f11" />

<img width="975" height="517" alt="image" src="https://github.com/user-attachments/assets/29de703a-280b-4bfa-b475-63b7f0b54143" />

**[fig 28]** — `prep` picking up `sky130_inv.lef` as an extra LEF, full synthesis completing at a new area figure.


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
<img width="975" height="450" alt="image" src="https://github.com/user-attachments/assets/41e6f5d1-65ef-45be-906f-8ba7de930abf" />

<img width="975" height="277" alt="image" src="https://github.com/user-attachments/assets/4c1d77dc-6245-474e-b9cb-b6ed7b36b6cd" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/cb8d8b22-b13f-4fd2-ba4e-2b2341f73e3f" />

**[fig 29]** — `init_floorplan` (die/core area extracted, DEF written), `place_io` (I/O pins placed), `tap_decap_or` (endcaps + tapcells inserted — 528 endcaps, 7872 tapcells this run), and the Magic view of the resulting floorplan.

### Post-synthesis timing analysis with OpenSTA

With the custom cell in the netlist, the real question was: what does it cost me in timing? Ran OpenSTA standalone against `pre_sta.conf` rather than trusting the flow's built-in check alone.

```tcl
set cmd_units -time ns -capacitance pF -current mA -voltage V -resistance kOhm -distance um
read_liberty -max .../src/sky130_fd_sc_hd__slow.lib
read_liberty -min .../src/sky130_fd_sc_hd__fast.lib
read_verilog .../results/synthesis/picorv32a.synthesis.v
link_design picorv32a
read_sdc .../src/my_base.sdc
report_checks -path_delay min_max -fields {slew trans net cap input_pin}
report_tns
report_wns
```

```bash
sta pre_sta.conf
```

<img width="975" height="521" alt="image" src="https://github.com/user-attachments/assets/b8d380f8-0290-4f48-b5fc-ac57b099af68" />

<img width="975" height="427" alt="image" src="https://github.com/user-attachments/assets/400d0579-5d8f-4f43-ba81-5cb56ff26287" />

**[fig 30]** — `pre_sta.conf` commands in the editor, and the actual `sta pre_sta.conf` run alongside: a hold-path check (`Path Type: min`) between two flip-flops on the custom `sky130_vsdinv` cell, closing at `0.24 slack (MET)`.

<img width="975" height="477" alt="image" src="https://github.com/user-attachments/assets/1a25b77e-3b14-4099-b35d-c72091fff886" />

**[fig 31]** — the same run's `report_tns` / `report_wns` output, landing at `tns -711.59` / `wns -23.89` — this is the setup-side view, and it's clearly still in violation.

### Reducing fanout and re-running synthesis

The custom inverter's un-optimized drive strength was causing high-fanout nets to slow down. Constrained max fanout and let synthesis re-map.

```tcl
prep -design picorv32a -tag 04-07-06-23 -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
set ::env(SYNTH_SIZING) 4
echo $::env(SYNTH_DRIVING_CELL)
run_synthesis
```

<img width="975" height="533" alt="image" src="https://github.com/user-attachments/assets/5b2f588b-2b92-4564-82fb-6366bdb86813" />

<img width="975" height="585" alt="image" src="https://github.com/user-attachments/assets/38325df3-7008-4839-aaee-94fbb0b1ada8" />

**[fig 32]** — `prep` re-run merging the custom LEF again, `SYNTH_DRIVING_CELL` confirmed as `sky130_fd_sc_hd__inv_8`, and the completed synthesis: new chip area **152,738.99 µm²**, closing at `tns 0.00` / `wns 0.00`.


| Run | Chip area | WNS |
|---|---|---|
| Day 4, first custom-cell synthesis | 181,729.29 µm² | (violating) |
| Day 4, after fanout/sizing rework | 152,738.99 µm² | 0.00 |

Interesting outcome — re-mapping with tighter sizing constraints actually brought the area *down* below the first custom-cell run, not just closed the timing. A different synthesis strategy chose noticeably different gate sizes for the same netlist.

### Timing ECO fixes (manual cell swaps)

Separately from the clean fanout-fixed run above, I went through the original (still-violating) netlist by hand to practice manual ECO — the kind of fix you'd do if re-synthesizing from scratch wasn't an option.

```tcl
report_net -connections _11643_
help replace_cell
replace_cell _14481_ sky130_fd_sc_hd__or4_4
report_checks -fields {net cap slew input_pins} -digits 4
```

<img width="975" height="613" alt="image" src="https://github.com/user-attachments/assets/f5c26a22-5669-43a1-a591-513ffebbdbe1" />

<img width="975" height="515" alt="image" src="https://github.com/user-attachments/assets/b587e414-5f81-4508-9edc-1a5aade8bb9c" />

**[fig 33]** — `report_net -connections _11643_` showing the driver (`_14481_/Y`, an OR gate of drive strength 2) feeding three loads it wasn't sized for; `replace_cell` swapping it for the drive-strength-4 version; timing report starting to regenerate.

<img width="975" height="596" alt="image" src="https://github.com/user-attachments/assets/7394bec0-b1ff-4cc2-8d42-5646907057ad" />

<img width="975" height="640" alt="image" src="https://github.com/user-attachments/assets/d75cb888-ceec-420c-b030-85d0b8dcdaea" />

**[fig 34]** — the regenerated timing report continuing through the same fanout chain, ultimately closing at `slack -23.8950 VIOLATED` for this particular path — better than before the swap, but this one path alone wasn't going to fix the whole design.


```tcl
report_checks -from _29052_ -to _30440_ -through _14506_
```

```bash
cd .../results/synthesis/
ls
cp picorv32a.synthesis.v picorv32a.synthesis_old.v
ls
```

<img width="975" height="783" alt="image" src="https://github.com/user-attachments/assets/7f8ea1d3-200f-4183-8508-e67b1d210cf8" />

**[fig 35]** — `report_checks -from -to -through` confirming the swapped instance now sits correctly in the path, followed by backing up the original netlist (`picorv32a.synthesis_old.v`) before anything gets overwritten.


I ended up carrying the **fanout-fixed 0/0 run** (fig 32) forward into floorplan/placement/CTS rather than the manually-patched netlist — the ECO swaps were a useful exercise in reading timing reports and using `replace_cell`, but re-synthesizing with the right constraints was the faster path to a clean design.

### Floorplan and placement on the clean netlist

```tcl
init_floorplan
tap_decap_or
run_placement
```

<img width="975" height="485" alt="image" src="https://github.com/user-attachments/assets/d6298cf3-3169-474b-a2b0-76f2bc8fa323" />

<img width="975" height="294" alt="image" src="https://github.com/user-attachments/assets/31e727d2-9484-4c15-8818-032f42ed8b29" />

**[fig 36]** — `init_floorplan` and `tap_decap_or` logs on the fanout-fixed netlist — endcaps and tapcells inserted, floorplan DEF written.

<img width="975" height="538" alt="image" src="https://github.com/user-attachments/assets/b01dfc3f-02da-4263-ba67-532edf40a932" />

**[fig 37]** — `run_placement` stats block.


| Metric | Value |
|---|---|
| Total instances | 10,915 |
| Fixed instances | 6,130 |
| Nets | 15,441 |
| Design area | 485,298.6 µm² |
| Fixed area | 8,041.0 µm² |
| Movable area | 141,947.4 µm² |
| Utilization | 36% |
| Utilization, padded | 55% |
| Rows | 234 |
| Row height | 2.7 µm |
| Original HPWL | 694,322.5 µm |
| Legalized HPWL | 707,253.4 µm |
| HPWL delta | +1.8% |

Worth flagging: HPWL actually got *worse* through legalization this time (+1.8%, vs. the −1.7% improvement back on Day 2). With the custom cell now in the mix, the legalizer had a slightly different set of legal sites to work with — not a red flag on its own, just a different placement problem than the stock-library-only run.

### Running CTS

```tcl
run_cts
```

<img width="975" height="406" alt="image" src="https://github.com/user-attachments/assets/8575d528-b9a3-481a-8ba1-0aacabc4b17f" />

<img width="975" height="465" alt="image" src="https://github.com/user-attachments/assets/c2ac837e-433b-4fbb-ad56-d7ce8fd6a3e9" />

**[fig 38]** — `run_cts` log: TritonCTS running, liberty trimmed, clock tree built, closing at `wns 0.00` / `tns 0.00`, with an initial (pre-buffer-list-change) clock skew report showing `Latency 4.44 ns`, `Skew 3.44 ns`.


### Post-CTS timing analysis in OpenROAD

```tcl
openroad
read_lef .../tmp/merged.lef
read_liberty $::env(LIB_SYNTH_COMPLETE)
read_verilog .../results/synthesis/picorv32a.synthesis_cts.v
link_design picorv32a
read_sdc .../src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4
```

<img width="975" height="367" alt="image" src="https://github.com/user-attachments/assets/4d66f349-3e30-4af3-8e1b-e2112af4448a" />

**[fig 39]** — the first attempt into OpenROAD hit a couple of ordering issues — `ORD-1018: no technology has been read` from trying to link before reading the LEF, and a `read_sdc` typo that killed the session with a segmentation fault. Re-ran the commands in the right order (LEF → liberty → verilog → link → sdc) and it went through cleanly.


<img width="975" height="488" alt="image" src="https://github.com/user-attachments/assets/4e7e34cf-a853-47fd-833c-e7a501938b37" />

**[fig 40]** — clean post-CTS `report_checks` output for a hold path, closing at `0.3819 slack (MET)`.

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/77f53fee-ccba-4a88-90d6-7d7441700a6c" />

**[fig 41]** — a second `report_checks` for a longer setup path through several `or2`/`o21a` cells, closing at `7.2027 slack (MET)`.

### Exploring `CTS_CLK_BUFFER_LIST`

Curious what removing the smallest clock buffer from the list would do to skew, so I pulled `sky130_fd_sc_hd__clkbuf_1` out and re-ran CTS.

```tcl
echo $::env(CTS_CLK_BUFFER_LIST)
set ::env(CTS_CLK_BUFFER_LIST) [lreplace $::env(CTS_CLK_BUFFER_LIST) 0 0]
set ::env(CURRENT_DEF) .../results/placement/picorv32a.placement.def
run_cts
```

<img width="975" height="523" alt="image" src="https://github.com/user-attachments/assets/50baec77-faf0-42b0-accc-df0ff399c677" />

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/b208a32a-8235-435b-8b87-563701146905" />

**[fig 42]** — a couple of `lreplace` syntax attempts that failed (`wrong # args`) before getting the bracket/quoting right, then the list confirmed without `clkbuf_1` (`clkbuf_2 clkbuf_4 clkbuf_8`), followed by CTS re-running successfully on the placement DEF.


```tcl
report_clock_skew -hold
report_clock_skew -setup
```

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/0a0746ee-89a2-4427-9f38-3213b1e919c8" />

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/80b54161-766b-46ba-a8ff-6dec87ad8fea" />

<img width="975" height="471" alt="image" src="https://github.com/user-attachments/assets/6c244454-31e0-4c89-beed-1708efa81604" />

**[fig 43]** — clock skew report with `clkbuf_1` removed: latency 1.67 ns / 0.91 ns on the two flops checked, CRPR 0.00, **skew down to 0.76 ns** for both hold and setup — a real improvement over the 3.44 ns skew from the original buffer list.


| Clock buffer list | Skew (hold) | Skew (setup) |
|---|---|---|
| Default (`clkbuf_1, 2, 4, 8`) | 3.44 ns | — |
| Without `clkbuf_1` | 0.76 ns | 0.76 ns |

Removing the smallest buffer forced CTS to build the tree out of larger, more consistent drive strengths — fewer stage-to-stage variations meant less accumulated skew. Small buffers are cheap on area but clearly not free on skew.

```tcl
set ::env(CTS_CLK_BUFFER_LIST) [linsert $::env(CTS_CLK_BUFFER_LIST) 0 sky130_fd_sc_hd__clkbuf_1]
echo $::env(CTS_CLK_BUFFER_LIST)
```

<img width="975" height="115" alt="image" src="https://github.com/user-attachments/assets/2594d62d-106a-4e36-a22e-a01f820dfe93" />

**[fig 44]** — putting `clkbuf_1` back at the front of the list and confirming it's restored, before moving on to Day 5.

---

## Day 5 — PDN, Routing, and Post-Route Sign-off

### Generating the Power Distribution Network

```tcl
prep -design picorv32a -tag 04-07-08-05 -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
run_synthesis
init_floorplan
tap_decap_or
run_placement
run_cts
gen_pdn
```

<img width="975" height="510" alt="image" src="https://github.com/user-attachments/assets/8ba5e480-8fe2-4120-a933-4e7ab9b9b189" />

<img width="975" height="521" alt="image" src="https://github.com/user-attachments/assets/33d71daa-0275-448b-b31d-890625a81108" />

**[fig 45]** — `gen_pdn` running: the same familiar vsrc-not-on-a-power-stripe warnings (still just the PDN step nudging via placements onto the nearest stripe, not an error), ending with `PDN generation was successful`.


### Viewing the PDN in Magic

```tcl
magic -T .../sky130A.tech lef read ../../tmp/merged.lef def read 14-pdn.def &
```

<img width="975" height="523" alt="image" src="https://github.com/user-attachments/assets/98d9bd78-363d-43c0-a328-6eaa2eb3e9a6" />

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/881cbc08-3138-4739-9cbe-50d2f1717814" />

**[fig 46]** — Magic loading the PDN DEF — a dense power mesh across the whole core — with tkcon printing a string of `Unknown keyword ... in LEF file, ignoring` messages (`PERIMETERSIDEAREA`, `ANTENNAMODEL`, `MAXIMUMDENSITY`, etc.). These are all just LEF fields Magic doesn't parse but doesn't need — harmless.


### Detailed routing with TritonRoute

```tcl
run_routing
```

<img width="975" height="325" alt="image" src="https://github.com/user-attachments/assets/cc84c425-b1c6-491b-84d0-52461915a74e" />

**[fig 47]** — routing kicking off: Global Routing stage, min routing layer 2 / max routing layer 6, unidirectional routing enabled, obstruction counts being processed layer by layer.


<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/53f84d3f-d03e-48ea-8a25-6c971a3a8018" />

**[fig 48]** — routing finishing up: parasitic extraction parameters (edge capacitance factor 1.0, wire model L), SPEF file written, and Static Timing Analysis kicked off automatically right after — closing with `Routing completed for picorv32a/04-07-09-35 in 15m36s`.


Worth calling out: the original workshop material treats parasitic extraction and post-route STA as two separate manual steps (an external Python SPEF extractor, then a standalone OpenROAD/OpenSTA session). On this run, `run_routing` chained RC extraction, SPEF writing, and STA automatically as part of the same step — so what was three tasks in the walkthrough became one continuous log for me. Also, 15 minutes for routing on this VM is a good reminder that detailed routing is genuinely the heaviest stage in the whole flow.

### Viewing the routed layout in Magic

```tcl
cd .../results/routing/
magic -T .../sky130A.tech lef read ../../tmp/merged_unpadded.lef def read picorv32a.def &
```

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/14527574-f34a-4d0e-a77e-98a36f78576d" />

**[fig 49]** — tkcon confirming the load: `Processed 52921 subcell instances total`, `427 pins total`, `15441 nets total`, `DEF read: Processed 356819 lines`.

<img width="975" height="521" alt="image" src="https://github.com/user-attachments/assets/c49c7ea2-09a8-4bb3-a6d3-6a18421ae76b" />

<img width="975" height="519" alt="image" src="https://github.com/user-attachments/assets/f499ee47-ccd0-4a9b-85ae-494a32763e68" />

**[fig 50]** — zoomed-in view of the fully routed layout — standard cells, filler cells, and decap cells packed into rows, with the actual metal routing tracks visible threading between them.


### Routing summary

| Metric | Value |
|---|---|
| Nets routed | 15,441 |
| Pins | 427 |
| Subcell instances | 52,921 |
| Min routing layer | 2 |
| Max routing layer | 6 |
| Wire model | L |
| Edge capacitance factor | 1.0 |
| Total routing runtime | 15m 36s |

That's the flow closed end-to-end — `picorv32a`, with a hand-built inverter cell wired into the standard cell library, taken from RTL through a fully routed, timing-clean layout.

---

## Acknowledgements

This log follows the VSD "Digital VLSI SoC Design and Planning" workshop by Kunal Ghosh (VSD Corp.), with the Day 3–5 custom standard cell lab built around Nickson P. Jose's `vsdstdcelldesign` reference design. All commands, screenshots, and numbers above are from my own runs on the workshop VM.

---

## Acknowledgements

This log follows the VSD "Digital VLSI SoC Design and Planning" workshop by Kunal Ghosh (VSD Corp.), with the Day 3–4 custom standard cell lab built around Nickson P. Jose's `vsdstdcelldesign` reference design.
