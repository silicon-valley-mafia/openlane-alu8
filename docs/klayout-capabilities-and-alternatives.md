# KLayout capabilities and alternatives for a Cadence Innovus flow on SCL CP8D

**Written:** 2026-11-25 · **Question answered:** *Can KLayout turn the SCL kit's inputs into neutral or Cadence-compatible files, and if not, what are the alternatives?*
**Companion docs:** [cadence-innovus-format-gaps.md](cadence-innovus-format-gaps.md) (what is missing), [echiphub-support-ticket.md](echiphub-support-ticket.md) (what to ask SCL), [cadence-flow-feasibility.md](cadence-flow-feasibility.md).

> This repo is public. No PDK file content is reproduced here; only file paths, layer numbers and ratios are quoted.

---

## 1. Short answer

| Need for Innovus | Can KLayout make it? | Notes |
|---|---|---|
| GDS / OASIS (neutral layout) | **Yes** | Native read/write of GDS2, OASIS, CIF, DXF |
| DEF+LEF → GDS merge (signoff stream-out outside Innovus) | **Yes** | LEF/DEF reader + `strm2gds`/`strm2oas`, driven by a layer map |
| Stream-out layer map | **Effectively yes** | The kit's `scl_cp8d.map` is already in the *Encounter/Innovus* map style (§3), so it only needs expanding |
| SPICE netlist from layout (for LVS / sim) | **Yes** | LVS writes SPICE (`target_netlist`) and L2N/LVSDB |
| LEF / DEF **writer** | **No** | KLayout imports LEF/DEF only |
| Tech LEF fixes (antenna, LEF 5.6+ rules) | **No** | Has to be hand-edited or come from SCL |
| QRC tech file / ICT / capTable | **No** | No KLayout path; needs the process stack (SCL) |
| SPEF for Tempus | **No** | KLayout `pex` (≥0.30.2) is resistance-only; KLayout-PEX (kpex) writes SPICE, not SPEF, and also needs the stack |
| Pegasus / PVS / Assura rule decks | **No** | KLayout decks are Ruby; no KLayout→PVL translator is mature (see §4) |
| Virtuoso `.tf` / `.drf` / OA library | **No** | `tf_import` goes the other way (Cadence → KLayout `.lyp`) |
| Antenna areas for cell LEF | **No** | Use Magic or Abstract Generator (§5) |

**Conclusion:** KLayout is the right **neutral bridge and signoff tool**. It reads what Innovus writes (GDS, DEF) and checks it with SCL's own decks. It **cannot create the missing Cadence technology files**; those need SCL data or other tools.

---

## 2. What KLayout reads and writes (sources: klayout.de docs, KLayout forum)

- **Layout formats:** GDS2 and OASIS (read/write), CIF and DXF (read/write), Magic `.mag` (read), LEF/DEF (read, with a map file), Gerber (import).
- **Command-line buddies:** `strm2gds`, `strm2oas`, `strm2cif`, `strmxor`, `strmclip`. These are useful for a scripted DEF+LEF → GDS merge and for XOR-comparing an Innovus GDS against a KLayout-merged GDS.
- **Verification:**
  - DRC/LVS scripts are written in a Ruby DSL (the SCL decks are `drc/cp8d_drc.drc` and `lvs/lvs_os_scl_cp8d.lvs`).
  - LVS compares the layout against a SPICE/CDL schematic (the kit provides `cdl/core_iolib_cp8d.cdl`).
- **Extraction:**
  - Built-in `pex` module (KLayout ≥ 0.30.2): resistance networks only.
  - Third-party **KLayout-PEX** (`pip install klayout-pex`, JKU, experimental): FasterCap/FastCap2/MAGIC/2.5D engines. It needs a process-stack description (the same data SCL has not given us) and writes SPICE netlists, not SPEF.
- **Cadence-adjacent import only:**
  - `tf_import` package: Cadence techfile + display file + layer map → KLayout `.lyp`.
  - Community scripts converting a Virtuoso layer map to `.lyp`.
  - There is no reverse path.

---

## 3. The stream-map finding: G6 is smaller than thought

The kit file `libs.tech/klayout/tech/scl_cp8d.map` uses KLayout's LEF/DEF layer-map syntax: `layerName purpose(s) gdsLayer gdsDatatype`.
- KLayout's author describes this syntax as the Cadence **Encounter** stream-out map format.
- KLayout issue #817 confirms it is the Encounter/Innovus `streamOut -mapFile` style, not the OA/DFII layer-map style.
- It is therefore **not a foreign format** for Innovus. It only needs:
  1. one object type per line (KLayout accepts comma lists like `NET,SPNET,PIN,VIA`; Innovus acceptance of comma lists is unverified);
  2. duplicate lines removed;
  3. `LABEL` replaced (it is not an Innovus purpose);
  4. optional `LEFOBS`/`FILL`/`DIEAREA` lines.

Draft Innovus map translated from the kit's layer numbers (metal1 = 8, via = 9, metal2 = 10; labels on datatype 7). **Unverified until it is round-tripped:**

```
# Innovus streamOut map draft for CP8D (derived from scl_cp8d.map)
# layerName  objectType  gdsLayer  gdsDatatype
metal1  NET     8  0
metal1  SPNET   8  0
metal1  PIN     8  0
metal1  LEFPIN  8  0
metal1  VIA     8  0
metal1  FILL    8  0
NAME    metal1/PIN     8  7
NAME    metal1/LEFPIN  8  7
via     VIA     9  0
via     LEFPIN  9  0
metal2  NET     10 0
metal2  SPNET   10 0
metal2  PIN     10 0
metal2  LEFPIN  10 0
metal2  VIA     10 0
metal2  FILL    10 0
NAME    metal2/PIN     10 7
NAME    metal2/LEFPIN  10 7
```

Notes:
- Front-end layers (well, diffusion, poly, contact) come from the cell GDS via `streamOut -merge std_cells_cp8d.gds pads_cp8d.gds CORNER.gds`, so they need no map lines.
- **Acceptance test:**
  1. Stream out from Innovus with this map.
  2. Merge DEF+LEF+cell GDS in KLayout with the original `scl_cp8d.map`.
  3. Run `strmxor` on the two GDS files: it must report 0 differences.
  4. Run SCL's KLayout DRC and LVS on the Innovus GDS.
- If Innovus is run with no `-mapFile`, it writes a template `streamOut.map`. Diff the template against this draft before the first real use.

---

## 4. Can the KLayout decks become Cadence (Pegasus/PVS) decks?

| Tool | Direction | State (as found 2026-11-25) |
|---|---|---|
| `pvcc-eda` (PyPI 0.1.3) | KLayout DRC ↔ SVRF / ICV / **Pegasus PVL** | Very early. Covers width, space, notch, enclosure, overlap, area, density and EOL only. Tested on sky130/FreePDK45 samples. No LVS. |
| PVconverter (GitHub, kiwa3927) | SVRF → PVS / PVRS | Needs a Calibre SVRF deck as input. We do not have one. |
| Commercial deck-porting services | Any → PVL | Possible, but a ported deck is not SCL-qualified. |

A translated Pegasus deck would be **our** deck, not SCL's. eChipHub's acceptance is based on SCL's KLayout DRC report. **Recommendation:** do not translate. Keep KLayout DRC/LVS as signoff and ask SCL (ticket items 8–9).

---

## 5. Alternatives, gap by gap

| Gap (see gap doc) | Alternative | Gives | Limits |
|---|---|---|---|
| **G1 extraction** (no QRC/capTable) | (a) **Correct the tech-LEF R/C locally**, using SCL's own Magic-tech extract coefficients (area + fringe per µm, per layer). Then run Innovus `extractRC` → `rcOut -spef` → Tempus. | SPEF with SCL-sourced coefficients | Coefficient model only, no 3-D coupling. Fine for a 2-metal 0.8 µm block. Label results "LEF-RC estimate". |
| | (b) **Hand-build an ICT** → `generateCapTbl` (Innovus utility). | A real capTable | Needs the dielectric stack, which only SCL can give. Back-solving heights from the Magic coefficients gives an internally inconsistent stack (εr 3.9: M1 ~1.2 µm, M2 ~2.5 µm above substrate, M2–M1 ~1.2 µm), so use it only as a sanity check. |
| | (c) **Magic `ext2spice`** on the final GDS, using SCL's Magic tech | R/C SPICE netlist from foundry numbers | Not SPEF. Good as a cross-check of (a). |
| | (d) **OpenRCX** | SPEF that Tempus can read | SCL's deck is all zeros, so useless until ticket item 2 is answered. |
| **G5 antenna** (no rules) | **Magic `lef write`** on cell GDS | Antenna gate/diffusion areas (`ANTENNAGATEAREA`/`ANTENNADIFFAREA`) for the cell LEF | `antennacheck` still needs ratios. SCL's Magic tech has none. |
| | **Cadence Abstract Generator** (`abstract`, part of the Virtuoso/IC install) | GDS → LEF with pins, obstructions and antenna areas | Needs the IC install on W0 (unchecked: `ls ~/cadence/installs`). Ratios are still needed from SCL. |
| | **Design-side mitigation** on a 2-metal process | Short M1 runs into gates, jumpers to M2 near gates, diode cells if the library has them | Engineering judgement, not signoff. Ask SCL for exemption or rules (ticket item 7). |
| **G6 stream map** | §3 translation | Innovus `streamOut -mapFile` | Needs an XOR round-trip check |
| **G7/G8 DRC/LVS decks** | KLayout DRC + LVS on the Innovus GDS. **Netgen** as a second LVS opinion. | SCL-accepted signoff | — |
| **G3/G4 LEF errors** (SITE 25.35 vs 32.35; CPERSQDIST ~1000× too large) | Local corrected copy of the tech LEF (never committed here) | Usable rows and sane RC | Must be confirmed by SCL (ticket items 3–5) |
| Virtuoso `.tf`/OA, Spectre | None for digital PnR | — | Not needed for Innovus. Out of scope. |

---

## 6. Recommended hybrid flow (until SCL replies)

```
Genus/Yosys netlist
   └─► Innovus  (LEF: corrected local tech LEF + kit cell/pad LEFs; .lib min/typ/max; MMMC without capTable)
          ├─ floorplan / place / CTS / route
          ├─ extractRC (LEF-RC) ─► rcOut -spef ─► Tempus  [label: estimate]
          └─ streamOut -mapFile cp8d_innovus.map -merge {cell, pad, corner GDS}
                 └─► KLayout:  strmxor vs KLayout-merged DEF  →  SCL DRC deck  →  SCL LVS deck (+ kit CDL)
                        └─► (optional) Magic ext2spice RC cross-check, Netgen LVS second opinion
```

In this flow, Innovus is used for implementation and **SCL's KLayout decks remain the signoff**. That is the arrangement asked about in ticket item 9.

---

## 7. Sources (plain links)

- KLayout docs, layer mapping / LEF-DEF import: https://www.klayout.de/doc/about/lef_def_import.html
- KLayout docs, buddy tools: https://www.klayout.de/doc/programming/ (strm2* tools)
- KLayout issue on Encounter map format: https://github.com/KLayout/klayout/issues/817
- KLayout forum, KLayout DRC vs Cadence/Calibre signoff: https://www.klayout.de/forum/discussion/1907
- KLayout-PEX: https://pypi.org/project/klayout-pex/
- pvcc-eda: https://pypi.org/project/pvcc-eda/
- PVconverter: https://github.com/kiwa3927/PVconverter
- Cadence forum, captables and `generateCapTbl` from an ICT: https://community.cadence.com/cadence_technology_forums/f/digital-implementation/11254/captables
- Cadence forum, ICT is normally foundry-made: https://community.cadence.com/cadence_technology_forums/f/custom-ic-design/20511/creating-ict-file
- Cadence forum, foundry gives only Calibre decks → ask the foundry: https://community.cadence.com/cadence_technology_forums/f/custom-ic-design/59266/adapt-cal-file-to-rul-file-for-pvs
- itf_to_ict → generateCapTbl chain (when a foundry ships only TLU+): https://cloud.tencent.com/developer/article/2187399
