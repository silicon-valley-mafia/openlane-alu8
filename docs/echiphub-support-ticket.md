# eChipHub / SCL support ticket — CP8D PDK collateral for a Cadence Innovus flow

**Written:** 2026-11-25 · **Kit:** `SCL_CP8D_PDK_v1` (open-source SCL 0.8 µm, digital + analog) · **Companion docs:** [cadence-innovus-format-gaps.md](cadence-innovus-format-gaps.md) (gap register G1–G11), [klayout-capabilities-and-alternatives.md](klayout-capabilities-and-alternatives.md)

Where to file: eChipHub portal → **HelpDesk** (registered login). Fallback contact listed on SCL's PDK page: info@scl.gov.in.
Rules used for this ticket: ask for *files or written confirmation*, not workarounds; one numbered item per gap so SCL can answer item-by-item; attach no PDK content (only file paths are quoted).

---

## A. Full ticket (copy-paste)

> **Subject:** CP8D (0.8 µm) PDK v1 — request for missing collateral to run a Cadence Innovus PnR flow, and three suspected tech-LEF errors
>
> Dear SCL / eChipHub PDK support team,
>
> We are implementing a small digital block on the SCL 0.8 µm CP8D process using the open-source PDK (`SCL_CP8D_PDK_v1`). The OpenLane + KLayout flow in the kit works for us. We would now like to place-and-route the same design in Cadence Innovus while keeping **your KLayout DRC/LVS decks as the signoff**. While setting this up we found the following items missing or inconsistent. Could you please confirm or provide each one?
>
> **Process data for parasitic extraction**
> 1. The kit has no Cadence extraction file (QRC tech file, ICT file or capacitance table), and the OpenRCX deck `libs.tech/openrcx/scl800.rcx.lib` contains only zero coefficients. Could you share either a QRC/ICT/capTable file, **or** the interconnect cross-section (thickness and dielectric constant of field oxide, poly–M1 dielectric, M1–M2 dielectric and passivation; thickness and sheet resistance of poly, metal1, metal2) so we can build a capacitance table ourselves?
> 2. Could you provide a populated OpenRCX deck for CP8D?
>
> **Tech LEF (`libs.tech/openlane/digital_cp8d/lef/tech_cp8d.lef`)**
> 3. `CAPACITANCE CPERSQDIST` for metal1/metal2 appears to be about 1000× larger than physically expected, and about 1000× larger than the area capacitance in your own Magic tech file (`analog_cp8d/.../libs.tech/magic/SCL_CP8D.tech`, *extract* section). Is this a unit error (fF vs pF)? Which value should be used?
> 4. `RESISTANCE RPERSQ` and the via resistance in the LEF differ from the sheet/contact resistances in the same Magic tech file. Which set is authoritative?
> 5. The `SITE` is 25.35 µm high but the standard cells in `lef/core` are 32.35 µm high. Is 32.35 µm the correct row height?
> 6. The LEF is version 5.3 with `ANTENNA`/`GEOMETRY` off and contains only width/spacing rules. Is an updated LEF (5.6 or later) with enclosure, minimum-area and antenna statements available?
>
> **Antenna**
> 7. The tapeout checklist asks for a DRC- and antenna-clean GDS, but we found no antenna rules in the DRC deck, the LEF or the Magic tech file. Please share the antenna ratios (gate area vs poly/metal1/metal2 area, and any diode credit), **or** confirm in writing that antenna checking is not required for CP8D.
>
> **Cadence-format collateral**
> 8. Does SCL provide, or plan to provide, any Cadence-format files for CP8D: an Innovus stream-out layer map, Pegasus/PVS/Assura DRC and LVS rules, a Virtuoso technology/display file, or Spectre models?
> 9. If none: is a GDS produced by Innovus (streamed out with the layer numbers in `libs.tech/klayout/tech/scl_cp8d.map`) acceptable for tapeout **provided it is clean on your KLayout DRC deck and KLayout LVS deck**? Is the KLayout DRC report the only report checked at submission?
>
> **Missing kit items**
> 10. `filler_placement.tcl` is referenced in the kit's installation instructions but is not included. Could you share it?
> 11. We could not find a seal-ring cell, a logo/layer-revision marker block or the tapeout submission form in the kit. Are these provided separately?
> 12. Are any SRAM / memory macros (LEF, GDS, Liberty, Verilog, CDL) available for CP8D or C1D?
>
> **Usage and schedule**
> 13. Is there any licence/NDA restriction on using this PDK with commercial Cadence tools (Innovus, Genus, Tempus)?
> 14. What are the upcoming MPW shuttle dates and submission deadline for CP8D?
>
> Exact file paths can be shared for any item. Thank you for your time and for making the kit openly available.
>
> Regards,
> Deepesh J
> *(college / organisation, eChipHub user ID)*

---

## B. Short version (for forms with a character limit, about 900 characters)

> CP8D PDK v1. For a Cadence Innovus PnR flow with KLayout DRC/LVS signoff we need:
> (1) a QRC/ICT/capTable file, or the dielectric/metal thickness and εr stack;
> (2) a populated OpenRCX deck (the current one is all zeros);
> (3) confirmation of the tech-LEF CPERSQDIST, which is about 1000× the area cap in your Magic tech file;
> (4) the row height: LEF SITE is 25.35 µm but cells are 32.35 µm;
> (5) antenna ratios, or written confirmation that an antenna check is not needed;
> (6) any Cadence collateral (stream map, Pegasus/PVS rules), or confirmation that an Innovus GDS is accepted if clean on your KLayout DRC+LVS decks;
> (7) the missing filler_placement.tcl, seal ring and submission form;
> (8) whether SRAM macros exist for CP8D/C1D;
> (9) licence terms for commercial tools;
> (10) the next CP8D shuttle date.
> Thank you.

---

## C. Tracking

| # | Item | Gap ID | Status |
|---|------|--------|--------|
| 1–2 | Extraction stack / RCX | G1, G2 | open |
| 3–6 | Tech LEF errors / version | G3, G4, G9 | open |
| 7 | Antenna rules | G5 | open |
| 8–9 | Cadence collateral / Innovus-GDS acceptance | G6–G8 | open |
| 10–12 | filler script, seal ring, form, SRAM | — | open |
| 13–14 | Licence, shuttle | — | open |

When SCL replies, update this table and record the reply date and ticket number here.
