# SCL CP8D (0.8 µm) in a Cadence / Innovus PD flow — format-gap reference

**Purpose.** A self-contained reference to carry into a conversation with SCL / eChipHub
officials or an industry PD engineer. It lists, file format by file format, what a Cadence
physical-design flow (Innovus first, the rest of the backend second) expects, what the SCL
CP8D kit actually ships, what breaks as a result, what we can work around ourselves, and the
exact question to ask for each gap.

Companion documents in this repo:
- `docs/cadence-flow-feasibility.md` — the full feasibility analysis (tool by tool, with the
  Innovus 25.13 User Guide quotes behind each claim).
- `docs/scl-pdk-caveats.md` — the PDK defect / caveat register from the OpenLane side.

---

## 0. Context in one screen

| Item | Value |
|---|---|
| Process / kit | SCL **CP8D, 0.8 µm**, 2-metal, 5 V — `SCL_CP8D_PDK_v1` (eChipHub open-source PDK) |
| Kit reference doc | CSCB03004Y, 114 pp, dated 11/09/26 |
| Kit's intended flow | Yosys / OpenLane (OpenROAD, OpenSTA, OpenRCX) / KLayout / Magic / xschem / Icarus |
| Cadence tools we have | Genus + Innovus (DDI251), Tempus, Xcelium, Conformal |
| What we proved on the open flow | alu8 RTL → routed GDS, router DRC 0, LVS clean (see `README.md`, `docs/signoff.md`) |
| Question being asked | Can the same kit drive a **Cadence Innovus PD flow**, and if not, which **files / formats** are missing? |

**How the gap was established.** Every file in the kit (102 directories / 264 files) was
scanned. A filename search for `qrc`, `tch`, `captbl`, `ict`, `assura`, `pegasus`, `pvs`,
`calibre`, `cds.lib`, `techfile`, `.tf`, `drf`, `virtuoso` returns **zero real hits** (the only
"PVS" matches are `PVSS01` ground-pad cells). The kit was packaged for the open-source flow;
only the format-neutral files are usable by Cadence.

---

## 1. Summary — what exists, what is missing

### 1.1 Shipped and directly usable by Cadence

| Format | File(s) in kit | Read by |
|---|---|---|
| Liberty `.lib` (NLDM timing/power) | `cp8d_core_{min,typ,max}.lib`, `cp8d_pads_{min,typ,max}.lib` | Genus, Innovus (MMMC), Tempus, Conformal |
| Cell LEF (abstracts) | `core_cp8d.lef`, `pads_cp8d.lef`, `corner_cp8d.lef` | Genus (physical), Innovus |
| Technology LEF | `tech_cp8d.lef` (**LEF 5.3**, Abstract Generator 5.5.12, 2004) | Innovus — *readable, but see Gap 3 and Gap 4* |
| Cell GDSII | `std_cells_cp8d.gds`, `pads_cp8d.gds`, `CORNER.gds` | Innovus `streamOut -merge` |
| Verilog simulation models | `cp8d.v` | Xcelium, Conformal |
| CDL netlists | cell `.cdl` | LVS (KLayout in our flow) |

### 1.2 Missing, or present only in a non-Cadence format

| # | Needed for | Cadence format expected | What SCL ships instead | Severity for Innovus PD |
|---|---|---|---|---|
| **G1** | RC extraction for timing (Innovus `extractRC`, Tempus SPEF) | **Quantus QRC tech file** (`qrcTechFile`, `.tch`) | OpenRCX deck `scl800.rcx.lib` — OpenROAD format, **all values zero** | 🔴 blocks accurate post-route timing |
| **G2** | RC extraction, the fallback path allowed above 32 nm | **Capacitance table** (`.capTbl`) generated from an **ICT process file** (`.ict`) | No `.ict`, no `.capTbl`; **dielectric stack data not published anywhere in the kit** | 🔴 same blocker as G1 — this is the root cause |
| **G3** | Antenna checking / repair (`verify_antenna`, NanoRoute diode insertion) | **LEF antenna statements**: `ANTENNAAREARATIO`, `ANTENNADIFFAREARATIO`… on layers; `ANTENNAGATEAREA`, `ANTENNADIFFAREA` on pins | Tech LEF generated with `[ ] Antenna` **off**; no antenna rules in **any** format (KLayout deck has none either) | 🔴 silent pass — and SCL requires an antenna-clean GDS |
| **G4** | In-tool DRC (`verify_drc`), router rule awareness | **LEF 5.6 / 5.7 rule constructs**: `CLEARANCEMEASURE`, `USEMINSPACING` (prerequisites), `MINWIDTH`, `MINSTEP`, `ENCLOSURE`, `ENDOFLINE`/`EOLSPACING`, `SPACINGTABLE`, `MAXVIASTACK`, … | LEF **5.3** with only `WIDTH`, `SPACING`, `MANUFACTURINGGRID` | 🟠 runs and reports **clean while checking almost nothing** |
| **G5** | Density checks (`verifyMetalDensity`, `verifyCutDensity`, fill) | LEF `DENSITY` / min-max density rules | None | 🟡 silent pass |
| **G6** | GDS export (`streamOut -mapFile`) | **Cadence stream layer map** (layer name · purpose · GDS layer · datatype) | `scl_cp8d.map` in **KLayout** format | 🟡 rebuildable by hand |
| **G7** | Row creation (`floorPlan -site`) | LEF `SITE` whose height equals the cell height | `SITE CORE … SIZE 3.900 BY 25.350`, but every cell is **32.35 µm** tall | 🟠 wrong value, not a format — wastes area or overlaps rails |
| **G8** | Signoff DRC / LVS in Cadence | **Pegasus** rule deck, **PVS** rule file, or **Assura** rules | KLayout Ruby decks (`.drc` v1.1, `.lvs`) | ⚪ not needed — eChipHub only accepts SCL's KLayout report |
| **G9** | Custom layout / viewing in Virtuoso | **Virtuoso technology file** (`.tf`) + **`display.drf`** | Magic `.tech`, KLayout layer properties | ⚪ not needed for a std-cell digital flow |
| **G10** | Cadence library database | **OpenAccess** libraries (lib / cell / view) + **`cds.lib`** / `lib.defs` | Flat LEF / GDS / CDL / Verilog | ⚪ Innovus runs from LEF without it |
| **G11** | Analog simulation in Spectre | Spectre model library (`.scs`) with process-corner sections | SPICE model cards (BSIM level-49) | ⚪ analog only — out of a digital PD flow |

**Bottom line for Innovus PD:** placement, CTS, routing and GDS streamout all run.
What cannot be trusted is **extraction (G1/G2)**, **antenna (G3)** and **in-tool DRC (G4/G5)**.
G1–G3 need data only SCL can supply; G4–G7 we can partly fix locally.

---

## 2. Each gap in detail

### G1 + G2 — No extraction technology (QRC `.tch` / ICT / capTbl)

**What Cadence expects.**
- Innovus and Tempus compute wire parasitics from a **Quantus QRC tech file** (`.tch`), a
  foundry-generated binary describing the full conductor/dielectric stack.
- The Innovus 25.13 UG (ch. 8, *Generating a Capacitance Table*) states that **above 32 nm**
  either a QRC tech file **or a capacitance table** is acceptable; below 32 nm QRC is mandatory.
  At 0.8 µm the capacitance-table route is the documented path.
- A capacitance table is generated from an **ICT file** + the tech LEF. If neither QRC nor a
  captable is supplied, Innovus builds a default table from generic heuristics — the UG's own
  words: *"this severely compromises accuracy."*

**What SCL ships.** No `.tch`, no `.ict`, no `.capTbl`. The only extraction deck is the
OpenROAD-format `scl800.rcx.lib` (2,763 B), in which **every resistance and capacitance entry
is zero**.

**What an ICT file needs vs. what the kit gives:**

| ICT input | In kit? | Source |
|---|---|---|
| Conductor min width / spacing | ✅ | tech LEF + DRC deck (m1 W 1.7 / S 1.2 µm; m2 W 2.0 / S 1.4 µm) |
| Conductor sheet resistance | ✅ | LEF `RPERSQ` m1 0.07, m2 0.04 Ω/□ + doc sheet-R table |
| Via resistance | ✅ | LEF `VIA M2_M1 … RESISTANCE 0.3375` |
| Conductor thickness | ⚠️ | LEF says `THICKNESS 1` for **both** metals — looks like a placeholder |
| Height above substrate | ⚠️ | LEF says `HEIGHT 1` for **both** metals — same concern |
| **Interlayer dielectric: constant (εr), thickness, height of each layer (field oxide, ILD, IMD, passivation)** | ❌ | **Not anywhere in the kit or its PDFs** |

**Related defect that makes the fallback dangerous.** The tech LEF declares
`CPERSQDIST` metal1 **0.027**, metal2 **0.011** pF/µm². That is ~1000× too large. Three
independent checks agree:
1. The sister C1D (1.2 µm) LEF from the same generator declares **0.000027** — same mantissa,
   `e-3` dropped.
2. Oxide physics: ε_ox / t_ox ≈ 3.45e-11 / 1e-6 = **3.45e-5 pF/µm²**.
3. SCL's own datasheet (CSCB03004Y §5.4) adds **0.00595 pF** routing cap to INVR00 — ≈ 24 µm
   of metal1 with the corrected value, ≈ 0.13 µm with the declared one.

**Effect on an Innovus run.** Post-route timing has no trustworthy wire RC. Our OpenLane runs
already prove this: a 1 mm die and a 4 mm die (16× area, ~40× wire length) give **identical**
timing reports to the picosecond — cell delay only.

**What we can do ourselves (estimate, never signoff).**
```tcl
setLayerRC -layer metal1 -capacitance 2.46e-4 -resistance 4.12e-5
```
(reads the declared 0.027 as fF/µm²). Every number produced this way must be labelled
*estimated RC*.

**Ask SCL:**
- **Q1.** Can you share the **cross-section / dielectric stack** for CP8D: thickness and εr of
  field oxide, ILD (poly–M1), IMD (M1–M2) and passivation, plus metal1/metal2 thickness and
  height above substrate? With these we can write the ICT and generate a capacitance table
  ourselves — no QRC file needed.
- **Q2.** Does SCL have (or plan) a **QRC tech file or ICT file** for CP8D, even under NDA?
- **Q3.** Is `CPERSQDIST 0.027 / 0.011` in `tech_cp8d.lef` a known unit error? What is the
  intended value?
- **Q4.** Why is the OpenRCX deck (`scl800.rcx.lib`) all zeros — is a populated version
  available? (The C1D deck has capacitance but also no resistance.)
- **Q5.** Are `THICKNESS 1` / `HEIGHT 1` in the tech LEF real values or placeholders?

### G3 — No antenna rules in any format

**What Cadence expects.** `verify_antenna` and NanoRoute antenna repair read **LEF antenna
statements**: process-antenna ratios on routing/cut layers (`ANTENNAAREARATIO`,
`ANTENNADIFFAREARATIO`, cumulative variants) and gate/diffusion area on every cell input/output
pin (`ANTENNAGATEAREA`, `ANTENNADIFFAREA`).

**What SCL ships.** The tech LEF header records how it was generated:
```
# Options:   [ ] Antenna
#            [ ] Geometry
#            [x] Technology
```
No antenna ratios on layers, no antenna areas on pins. The KLayout DRC deck also contains no
antenna rule; a search of the whole kit and all five SCL documents for `antenna` returns zero
hits.

**Effect.** `verify_antenna` runs and reports nothing. OpenLane's `GRT_REPAIR_ANTENNAS` /
antenna check behaves identically. Two independent toolchains, same silent pass, same cause.
Meanwhile the submission requirement asks for a **"DRC/Antenna clean GDSII"**.

**Ask SCL:**
- **Q6.** What are the **antenna rules** for CP8D (max poly/metal1/metal2 area ratio to gate
  area, cumulative or per-layer, diode protection credit)?
- **Q7.** Or is a 2-metal 0.8 µm process **antenna-exempt** for MPW submission? If so, can that
  be stated in writing so the submission checklist is satisfied?
- **Q8.** Can a tech LEF regenerated **with the Antenna option on** (and pin
  `ANTENNAGATEAREA` / `ANTENNADIFFAREA`) be provided?

### G4 + G5 — Tech LEF is version 5.3 with almost no rule content

**What Cadence expects.** Innovus `verify_drc` uses only the rules **in the tech LEF**
(UG ch. 9). It needs two prerequisites — `CLEARANCEMEASURE` and `USEMINSPACING` — and reads
constructs such as `MINWIDTH`, `MINSTEP`, `ENCLOSURE`, `ENDOFLINE`, `EOLSPACING`,
`SPACINGTABLE`, `ADJACENTCUTS`, `MAXVIASTACK`. Density checks need density rules in the LEF.

**What SCL ships.** `tech_cp8d.lef`, **LEF 5.3** (2004), containing per layer only `TYPE`,
`DIRECTION`, `PITCH`, `OFFSET`, `WIDTH`, `SPACING`, `RESISTANCE`, `CAPACITANCE`,
`EDGECAPACITANCE`, `HEIGHT`, `THICKNESS`, `WIREEXTENSION`; one `VIA`, one `VIARULE GENERATE`,
five `SITE`s. Neither prerequisite, none of the advanced constructs, no density rules.

**Effect.** `verify_drc` checks min width, basic spacing and the via — and returns **clean**.
That is more dangerous than an error: it looks like signoff. The KLayout deck's
**71 rule categories** (well, boron, poly extension, contact enclosure, pad spacing, …) are
front-end rules LEF cannot express at all. Also documented but checked by **neither** LEF nor
the KLayout deck: **maximum metal width 25 µm** (CSCB03004Y §10.5.1) — wide power straps pass
and are still non-compliant.

**What we can do.** Treat `verify_drc` as a routing sanity check only; signoff stays KLayout.
`verifyConnectivity` (needs only the DEF) is genuinely useful.

**Ask SCL / expert:**
- **Q9.** Is a **newer tech LEF (5.6/5.7)** with enclosure, min-step, end-of-line and density
  rules available, or planned?
- **Q10.** Should the 25 µm **max metal width** rule be enforced by the KLayout deck (it isn't
  in v1.1), and should power straps be slotted?

### G6 — Layer map only in KLayout format

**What Cadence expects.** `streamOut -mapFile` takes a Cadence stream map: one line per
layer/purpose with GDS layer number and datatype (e.g. `metal1 drawing 8 0`, `metal1 pin …`,
`metal1 label 8 7`).

**What SCL ships.** `libs.tech/klayout/tech/scl_cp8d.map` (KLayout format). The same
layer/datatype pairs are recoverable from it and from the DRC deck, so translation is mechanical.

**Also note.** `streamOut -merge` with the three PDK GDS files: on a cell-name collision the
first file wins and the second is silently dropped unless `-uniquifyCellNames` is used.

**Ask SCL:**
- **Q11.** Can you confirm the **pin and label purposes / datatypes** expected by the KLayout
  LVS deck (e.g. is metal1 label 8/7 the only text layer), so our Cadence map produces a GDS
  the deck accepts?

### G7 — LEF site height ≠ cell height

**What Cadence expects.** `floorPlan -site CORE` builds rows at the SITE height; cells must be
exactly that tall (or an integer multiple).

**What SCL ships.** `SITE CORE … SIZE 3.900 BY 25.350`, while every cell in the datasheet
(Annexure-2) is **32.35 µm** tall. SCL's own OpenLane script works around it by deleting every
odd row after floorplan, which wastes about 36 % of the core area. Innovus has no direct
equivalent; options are `deleteRow` after `floorPlan`, or editing SITE to 32.35 in a local LEF
copy.

**Ask SCL / expert:**
- **Q12.** Is 25.35 deliberate — i.e. are rows meant to **alternate N / FS with shared rails**
  at a 25.35 pitch — or should the SITE be 32.35?
- **Q13.** If 32.35 is correct, is a corrected tech LEF available?

### G8 — No Cadence signoff deck (Pegasus / PVS / Assura)

Not a request. eChipHub requires the DRC report from **SCL's own KLayout deck**
(`Silicon no._Design Name_Date_Version_drc.txt`), so physical-verification signoff is KLayout
by specification. A hand-translated Pegasus deck would carry no foundry authority. Cadence can
produce the GDS; KLayout signs it off.

- **Q14.** Is a GDS produced by **Innovus** (rather than OpenLane) acceptable for submission,
  provided the KLayout DRC/LVS reports are clean?

### G9 – G11 — Virtuoso techfile, OpenAccess, Spectre models

Needed only for custom/analog work or for opening the design in Virtuoso. Not on the critical
path of a standard-cell Innovus flow. If ever needed:
- **Q15.** Is a Virtuoso `.tf` / `display.drf` / OA library set for CP8D available, or is
  building one from the KLayout layer data acceptable to SCL?

---

## 3. Two non-format questions that gate the whole Cadence route

- **Q16 — Licensing.** The DRC deck header marks the kit as provided under SCL NDA for
  evaluation / internal use, no redistribution. Is using the kit inside commercially licensed
  Cadence tools on our own workstation within that permission?
- **Q17 — Missing kit pieces already referenced by SCL's own documents.** The README mandates
  `filler_placement.tcl` for combinational designs (not shipped; `3_src` has 10 scripts). The
  MPW page says a **seal ring / layer-revision block** is "given in the PDK" (not found). The
  **Tapeout Submission Form** is referenced four times (not found). Where can these be
  obtained?

---

## 4. Questions for an industry PD engineer (not SCL-specific)

1. For a 2-metal mature node with no QRC file, is a **hand-written ICT → capTbl** the normal
   practice, and how far off is it typically versus a foundry QRC deck?
2. Is `verify_drc` with a thin LEF ever used as more than a routing sanity check in practice,
   or does everyone rely on the external signoff deck only?
3. With only two metals, how would you plan the **power grid** in Innovus (metal1 rails +
   horizontal straps, no vertical metal2 stripes) and still close routing?
4. How would you **validate** an Innovus-generated GDS against a KLayout-only signoff deck —
   is a stream map + KLayout DRC/LVS on the merged GDS sufficient?
5. Is running Innovus and OpenROAD **side by side on the same netlist** (timing, congestion,
   `verify_drc` vs KLayout DRC) a worthwhile cross-check to report?

---

## 5. What we already know works (so the conversation starts from facts)

| Step | Status on CP8D in Cadence |
|---|---|
| Genus synthesis with the shipped Liberty | ✅ |
| Innovus import (tech + cell LEF, MMMC with min/typ/max libs) | ✅ |
| Floorplan / place / CTS / route | ✅ — with the G7 site fix |
| `verifyConnectivity` | ✅ meaningful |
| `verify_drc`, `verify_antenna`, density checks | ⚠️ run, check almost nothing (G3–G5) |
| Post-route timing | ⚠️ cell delay real, wire RC estimated only (G1/G2) |
| `streamOut` to GDS | ✅ — with a translated map (G6) |
| Signoff DRC / LVS | KLayout only (G8), as on the OpenLane path |

**One-line summary to open with:** *the CP8D kit lets Innovus build the chip, but not judge it —
we need the dielectric stack (for extraction), the antenna rules, and confirmation of the
row/site height; everything else we can handle ourselves or is signed off in KLayout anyway.*
