README_for_Using_the_YD-RP2040

This README explains three critical YD-RP2040 ↔ Raspberry Pi Pico pin differences (pins 35, 36, 37), why they matter, and how to fix them in-circuit using the YD jumpers and the project board's cuttable pads.

---

**Overview**

Most Pico header pins match on the YD-RP2040 (GPIO0–22, 26–28, VSYS, VBUS, GND). However, the YD board intentionally remaps some signals to expose extra features. The three pins you must watch for when inserting a YD into a project board that expects Pico behavior are the physical header pins 35, 36 and 37.

**Pin mapping differences**

- Pin 37 (major mismatch)
  - Pico expected: `3V3_EN` (input that enables/disables the board's onboard 3.3V regulator).
  - YD-RP2040: `GPIO23` (also routed to the on-board RGB LED via `JP2`).

- Pin 35
  - Pico expected: `ADC_VREF` (external ADC reference input used by A/D conversions).
  - YD-RP2040: `GPIO29` (also available as `ADC3`).

- Pin 36
  - Pico expected: `3V3 OUT`.
  - YD-RP2040: usually labeled `+3V3` on the schematic (remains a 3.3V output on most versions); confirm with your board silkscreen/schematic if it might instead be `VSYS`.

Why this can break a project board

- If the project board expects pin 37 to be `3V3_EN`, but the YD header drives `GPIO23` there, the microcontroller can inadvertently toggle the regulator enable or be damaged by conflicting drivers. The RGB LED tied to `GPIO23` may also present a load or short.
- If the project board drives an ADC reference into pin 35, and the YD is presenting `GPIO29` (which may not be high-impedance), the reference may be shorted or corrupted.

---

**In-circuit fixes using available jumpers/pads**

The provided YD schematic exposes three useful controls for these issues: `JP1` (ADC VREF tie to 3.3V), `JP2` (RGB LED / `GPIO23` connection), and the project board's per-pin cuttable pads for the side header. Use only the changes listed below — they avoid a PCB re-spin.

Fix A — Resolve Pin 37 (`3V3_EN` vs `GPIO23`)

1. Power down the system and remove all power.
2. On the project board: locate the cuttable pad/jumper for physical pin 37 (the pad that connects the project-side `3V3_EN` net to the mating header). Carefully cut/open that jumper so the project board's `3V3_EN` rail is no longer directly driven by the YD header.
3. On the YD board: open `JP2` to disconnect the on-board RGB LED from `GPIO23` if you plan to use `GPIO23` on the header.
4. If the project board needs `3V3_EN` asserted to enable its regulator, restore the logic-high by adding a small wire or pull-up on the project board — tie the board-side pad (the pad that is now isolated after cutting) to `+3V3` or add a ~2.2k pull-up to `+3V3`. This permanently enables the 3.3V rail while leaving `GPIO23` free on the YD.

Why: Cutting isolates the YD header pin from the project's regulator enable net; opening `JP2` isolates the RGB LED. A manual tie or pull-up restores the expected enable function safely.

Fix B — Resolve Pin 35 (`ADC_VREF` vs `GPIO29/ADC3`)

1. On the YD board: close `JP1` to connect the RP2040 ADC reference internally to `+3V3` (so the RP2040 uses the board 3.3V as ADC_VREF). This is recommended when you don't need a precision external reference.
2. On the project board: locate and cut/open the pad/jumper that routes the project's external `ADC_VREF` to pin 35. This prevents the project board from driving the ADC reference into the YD's `GPIO29`.

If instead you need the project board's external precision reference available to the RP2040:
- Leave the project pad closed (so the external VREF reaches the header) and open `JP1` on the YD so the RP2040 ADC reference is not forced to `+3V3`. Buffer the external reference on the project board (op-amp or high-impedance driver) before connecting to the header to avoid contention with `GPIO29`.

Why: Closing `JP1` gives the RP2040 a stable internal ADC reference and cutting the project jumper prevents contention. If you need the external VREF, do the opposite and buffer it.

Fix C — Verify Pin 36 (`+3V3`)

- Inspect the YD silkscreen/schematic to confirm pin 36 is `+3V3`. On most YD schematics this is a 3.3V output. No jumper modifications are normally required for pin 36 unless your project explicitly expects `VSYS`.

---

Recommended minimal-change procedure (what I recommend)

Goal: Preserve the project board's control of `3V3_EN` while preventing the YD-RP2040 from unintentionally driving that net, and avoid modifying the project board unless necessary.

Summary of changes (minimal, reversible):
- On the YD board: close `JP1` (tie ADC VREF internally to `+3V3`) so the RP2040 uses the board 3.3V as ADC reference (safe default).
- On the YD board: open the per-pin jumper that connects `GPIO23` (RP2040 pin) to the physical header pin 37 and open/cut that jumper (this isolates the RP2040 from the header so it cannot drive the project's `3V3_EN`).
- On the YD board: open `JP2` to disconnect the onboard RGB LED from `GPIO23` (prevents LED load on the isolated GPIO).
- Do not cut the project board jumpers by default. Only cut the project-side pad for pin 35 or 37 if you have a specific reason (external precision VREF, or you want the project to permanently tie `3V3_EN` high).

Step-by-step (do this order)
1. Power down and disconnect all supplies.
2. On the YD board: close `JP1` (solder the jumper) to set RP2040 ADC VREF = `+3V3`.
3. On the YD board: locate the small per-pin solder jumper that connects RP2040 `GPIO23` to header pin 37 and open/cut that jumper (use a hobby knife or desoldering braid to remove the solder bridge). This prevents `GPIO23` from reaching the header.
4. On the YD board: open `JP2` (remove the jumper or cut its pad) to disconnect the RGB LED from `GPIO23`.
5. Verify visually that the YD-side solder jumper for pin 37 is open and `JP2` is open; check continuity between RP2040 `GPIO23` pad and header pin 37 (should be open).
6. Verify `ADC_VREF` on the YD is tied to `+3V3` (check continuity between `ADC_VREF` pad and `+3V3`). If you require an external precision VREF later, reverse `JP1` and cut the project pad for pin 35 instead.
7. Verify `+3V3` on pin 36 is present and correct.
8. Power up and test functionality.

Why this is minimal and safe
- By opening the YD-side jumper for pin 37 we avoid modifying the project PCB; the project retains control of `3V3_EN` via its own circuitry.
- Closing `JP1` prevents external ADC reference contention without changing the project PCB. If you later need the project's external VREF, you can open `JP1` and keep the project pad closed.
- Opening `JP2` removes the LED load so `GPIO23` (if reconnected later) won't accidentally light or load the project net.

Tools and verification
- Tools: fine-tip soldering iron, solder wick, flux, small hobby knife or soldering probe, multimeter.
- Test: continuity checks with DMM before power-up; measure `+3V3` after power-up.