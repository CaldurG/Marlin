# Ender 3 (SKR 1.3) — Marlin 2.1.x Configuration Plan

If something in configs is changed, leave default value in a comment.

Ported from the working `skr1.3` branch (Marlin 2.0.x, 2022) to `bugfix-2.1.x`
on branch `skr1.3_marlin2.1`. Hardware: Ender 3 + BTT SKR 1.3, TMC2208 (UART),
BLTouch clone (3D Touch), Titan direct drive, CR10 stock display, SD printing.

## Configuration.h

### Identity & serial

- `CUSTOM_MACHINE_NAME "Ender-3"`

### Stepper drivers (UART mode)

- `X_DRIVER_TYPE TMC2208`
- `Y_DRIVER_TYPE TMC2208`
- `Z_DRIVER_TYPE TMC2208`
- `E0_DRIVER_TYPE TMC2208`

### Motion

- `DEFAULT_AXIS_STEPS_PER_UNIT { 80, 80, 400, 830 }` (E: ~830 for Titan direct
  drive with the new LDO-42STH25-1404MAC 0.9° motor — exactly double the 1.8°
  value of 415 (0.9° = 400 full steps/rev); calibrate after flashing)
- `DEFAULT_MAX_FEEDRATE { 100, 100, 5, 50 }`
- `DEFAULT_MAX_ACCELERATION { 500, 500, 100, 5000 }`
- `DEFAULT_ACCELERATION 500`
- `DEFAULT_RETRACT_ACCELERATION 500`
- `DEFAULT_TRAVEL_ACCELERATION 500`
- `JUNCTION_DEVIATION_MM` — leave the default (0.013); the preferred value is
  set at runtime from the slicer start G-code (see "After flashing")
- `S_CURVE_ACCELERATION` (uncomment)

### Directions

- `INVERT_X_DIR true`
- `INVERT_Y_DIR true`
- `INVERT_Z_DIR false`
- `INVERT_E0_DIR false`

### Geometry

- `X_BED_SIZE 225`
- `Y_BED_SIZE 235`
- `Z_MAX_POS 250`

### Probe (BLTouch)

- `BLTOUCH` (uncomment)
- Keep `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN`
- `Z_MIN_PROBE_ENDSTOP_HIT_STATE` — leave the default (HIGH); in 2.1.x
  BLTouch requires HIGH (the old 2.0.x
  `Z_MIN_PROBE_ENDSTOP_INVERTING true` is handled internally by the
  BLTouch driver; the sanity check errors out on LOW)
- `NOZZLE_TO_PROBE_OFFSET { -42, -5, 0 }`
- `MIN_PROBE_EDGE 20`
- `XY_PROBE_FEEDRATE` — leave the default (133*60 mm/min); old config
  had 5000, the 2.1.x default is faster anyway (renamed from
  `XY_PROBE_SPEED`)
- `Z_MIN_PROBE_REPEATABILITY_TEST` (uncomment)
- `PROBING_HEATERS_OFF` (uncomment)
- `PROBING_FANS_OFF` (uncomment) — part-cooling fan off during probing;
  fan vibration/EMI can cause false triggers on BLTouch clones
- `PROBE_OFFSET_WIZARD` (uncomment, Configuration_adv.h ~line 1616) —
  LCD-guided probe Z-offset calibration: probes the bed, then a paper
  test at the same spot, and sets M851 Z automatically
- `Z_CLEARANCE_FOR_HOMING 4` (old name: `Z_HOMING_HEIGHT`)
- `Z_AFTER_HOMING 10` (uncomment, ~line 1938) — lift Z to 10 mm after
  homing instead of parking at the bed; easier nozzle access and less
  probe-collision risk
- `Z_SAFE_HOMING` (uncomment; recommended with probe homing)
- `NO_MOTION_BEFORE_HOMING` (uncomment, ~line 1925) — inhibit G-code
  moves until all axes are homed; safety net against SD files that
  assume a known position

### Bed leveling

- `AUTO_BED_LEVELING_3POINT` (uncomment) — flat glass bed: a single
  tilted plane is all that's needed (bilinear mesh would be overkill,
  and its 3×3 grid puts probe points under the 15 mm glass clips on
  the X middle). 3 corner points, none under the clips.
- `PROBE_PT_1 { 35, 205 }`, `PROBE_PT_2 { 35, 30 }`,
  `PROBE_PT_3 { 180, 117 }` (Configuration_adv.h ~line 2551) —
  back-left, front-left, right-middle; an even triangle across the
  bed, clear of the clips, within probe reach (X ≤ 183 due to the -42
  probe offset)
- `ENABLE_LEVELING_FADE_HEIGHT` — not needed with 3-point: the
  correction is a flat tilted plane, valid at any Z (fading it would
  drift tall prints off-plane). It's also inside a mesh-only
  conditional block, so it's compiled out automatically
- `LCD_BED_LEVELING` — not needed: with an automatic probe the
  "Probe and Level" menu (Level Bed = G29, on/off toggle) is provided
  by `HAS_LEVELING` alone; `LCD_BED_LEVELING` only adds the guided
  procedure for *manual* leveling (PROBE_MANUALLY / MESH_BED_LEVELING)
- `ASSISTED_TRAMMING` (uncomment, Configuration_adv.h ~line 1125) —
  G35 probes the bed corners and reports which screw to turn and how
  much. Set `TRAMMING_POINT_XY { { 35, 30 }, { 183, 30 }, { 183, 205 },
  { 35, 205 } }` (35 mm inset on X, 30 mm on Y, for the 225×235 bed;
  right points capped at X=183 — the probe offset of -42 means probing
  further right would push the nozzle past X_MAX_POS) and
  `TRAMMING_SCREW_THREAD M3_CW` (Ender 3 thumbwheels)

### Babystepping

- `BABYSTEPPING` (uncomment, ~line 2425) — nudge Z in tiny steps live
  during the first layer
- `DOUBLECLICK_FOR_Z_BABYSTEPPING` (uncomment) — double-click the
  encoder on the status screen to open babystepping
- `BABYSTEP_ZPROBE_OFFSET` (uncomment) — babysteps also adjust the
  probe Z offset (M851), so the fix persists after `M500`

### Thermal

- `MPCTEMP` (uncomment, ~line 715) — Model Predictive Control replaces
  PID for the hotend (PID was overshooting). Keep `PIDTEMP` commented
  (they are mutually exclusive). Requires:
  - `MPC_AUTOTUNE` (uncomment) — `M306 T` measures the hotend model
  - `MPC_HEATER_POWER { 40.0f }` — Ender 3 stock 40 W cartridge
  - `MPC_INCLUDE_FAN` (default on) — models part-fan cooling
  - `MPC_AUTOTUNE_MENU` (uncomment, ~line 741) — run `M306 T` from the
    LCD (no host available)
  - After flashing: run `M306 T` once at ~215 °C, then `M500`
  - Set filament heat capacity per material if needed:
    `M306 H0.0056` (PLA), `H0.0036` (PETG)
- `PIDTEMPBED` (uncomment, ~line 825) — PID for the bed instead of
  bang-bang; tighter temp control. The stock `DEFAULT_BED_*` values are
  generic — autotune after flashing (see After flashing step 3).
  `PID_AUTOTUNE_MENU` (uncomment, ~line 934) covers the bed too once
  `PIDTEMPBED` is on (Bed PID Autotune appears in the LCD Temperature
  menu).
- `HEATER_0_MAXTEMP 300` (stock 275; old config 290) — the thermistor
  table (EPCOS 100k, type 1) goes up to 320 °C; 300 leaves headroom
  under the table limit while covering any filament you'd run through
  a stock Ender 3 hotend
- `PID_EDIT_MENU` / `PID_AUTOTUNE_MENU` — DROP (PID no longer used for
  the hotend; M303 does not apply to MPC)
- Preheat presets — DISABLED in the menu: `PREHEAT_1_LABEL` /
  `PREHEAT_2_LABEL` commented out (~line 2568). With no LABEL defined,
  `PREHEAT_COUNT` becomes 0 and all preheat menu items disappear
  (Temperature menu, Configuration → Preheat Settings, filament-change
  presets, UBL preheat). The `PREHEAT_*_TEMP_*` values stay defined
  (unlabeled) because `PID_AUTOTUNE_MENU` uses `PREHEAT_1_TEMP_BED`
  (55 °C) as the bed autotune default. Set hotend/bed temps directly
  in the Temperature menu instead.

### Features

- `EEPROM_SETTINGS` (uncomment)
- `EEPROM_INIT_NOW` (uncomment, ~line 2537) — init EEPROM automatically
  on the first boot after flashing a new build; replaces the manual
  `M502` + `M500` step after each re-flash during tuning
- `SDSUPPORT` (uncomment)
- `CR10_STOCKDISPLAY` (uncomment)
- `NOZZLE_PARK_FEATURE` (uncomment)
- `NUM_SERVOS 1` (uncomment; BLTouch servo)
- Buzzer off (leave `LCD_FEEDBACK_FREQUENCY_*` unset)
- `INDIVIDUAL_AXIS_HOMING_MENU` — leave OFF (old config had it; not
  wanted, "Home All" is enough)
- `LCD_INFO_MENU` — leave OFF (considered; read-only info pages, not
  needed)

## Configuration_adv.h

### SD card

- `SDCARD_CONNECTION ONBOARD` (uncomment, ~line 2084) — the SD card in
  the **onboard slot on the SKR 1.3 board** is used, not the LCD's slot.
  Without this the pins default to the LCD card reader and the onboard
  card won't mount.
- `SD_DETECT_STATE HIGH` — DROP this entry: with `SDCARD_CONNECTION
  ONBOARD` on LPC1768, the firmware undefines `SD_DETECT_PIN` entirely
  (the onboard slot is shared as a USB mass-storage drive; the LCD gets
  mount/unmount menu entries instead of card detect). The old config's
  `SD_DETECT_INVERTED` applied to the LCD-slot wiring and is not needed.
- `SDCARD_SORT_ALPHA` (uncomment, ~line 1937) — sort SD file list
  alphabetically instead of FAT order; enable `SDSORT_USES_RAM` for
  fast sorting (LPC1768 has the RAM for it)
- `SD_CHECK_AND_RETRY` (uncomment, Configuration.h ~line 2784) — CRC
  checks and retries on SD reads; cheap insurance for SD-only printing
- `SCROLL_LONG_FILENAMES` (uncomment, ~line 1955) — scroll long
  filenames in the SD menu instead of truncating them

### Pause & park

- `ADVANCED_PAUSE_FEATURE` (uncomment, ~line 3098) — makes LCD/SD pause
  (M125) retract, lift Z, park the head, and hold temperatures
- `PARK_HEAD_ON_PAUSE` (uncomment) — actually move to the park position
  during pause (uses NOZZLE_PARK_FEATURE already in the plan)
- Skip `FILAMENT_LOAD_UNLOAD_GCODES` — no M701/M702 or LCD
  Load/Unload menu (M600 is compiled in but inert without a runout sensor)

### Linear Advance

- `LIN_ADVANCE` (uncomment, ~line 2483) — required for `M900` to work;
  without it the slicer's `M900 K0.04` is silently ignored
- `ADVANCE_K 0.04` (renamed from `LIN_ADVANCE_K` in 2.1.x) — firmware
  default; the slicer also sets it at runtime via `M900 K0.04`
- Compatible with `S_CURVE_ACCELERATION` (no conflict in 2.1.x; LA
  piggy-backs on the same ISR values)

### TMC2208 UART settings

- `HOLD_MULTIPLIER 0.5`
- `INTERPOLATE true`
- `X_CURRENT 600`, `X_MICROSTEPS 16`, `X_RSENSE 0.11`
- `Y_CURRENT 600`, `Y_MICROSTEPS 16`, `Y_RSENSE 0.11`
- `Z_CURRENT 600`, `Z_MICROSTEPS 16`, `Z_RSENSE 0.11`
- `E0_CURRENT 800`, `E0_MICROSTEPS 16`, `E0_RSENSE 0.11`
  — motor: LDO-42STH25-1404MAC (Slim Power pancake, 0.9°, rated 1.4 A/phase,
  2.4 Ω, 5.5 mH, holding torque ≥18 N·cm, CW rotation; spec sheet:
  <https://www.bondtech.se/wp-content/uploads/2019/08/SLIM_POWER_LDO-42STH25-1404MAC_RevA.pdf>).
  800 mA = 57% of rated — safe; headroom to ~1.1 A (85%) if the Titan skips
  steps. Torque is low (pancake) — if under-extrusion appears, raise current
  first. CW motor: flip `INVERT_E0_DIR` if it extrudes backwards
- `STEALTHCHOP_XY` (uncomment; Z and E stay spreadCycle, as before)
- `CHOPPER_TIMING CHOPPER_DEFAULT_24V`
- `MONITOR_DRIVER_STATUS` (uncomment) with:
  - `CURRENT_STEP_DOWN 50`
  - `REPORT_CURRENT_CHANGE`
  - `STOP_ON_ERROR`
- `HYBRID_THRESHOLD` (uncomment) with:
  - `X_HYBRID_THRESHOLD 100`
  - `Y_HYBRID_THRESHOLD 100`
  - `Z_HYBRID_THRESHOLD 5`
  - `E0_HYBRID_THRESHOLD 20`
- `TMC_DEBUG` (uncomment)
- Sensorless homing: leave OFF (TMC2208 has no StallGuard)
- `STATUS_MESSAGE_SCROLLING` (uncomment) — scroll long status messages
  on the 16-column LCD instead of truncating them

### Homing

- `QUICK_HOME` (uncomment, ~line 965) — home X and Y with a diagonal
  move in one pass instead of sequentially (faster homing, as in the
  old config)

### Buffers

- `BUFSIZE 16` (stock 4, ~line 2807) — G-code command ring buffer;
  larger buffer keeps the planner fed during SD prints
- `BLOCK_BUFFER_SIZE 32` (stock 16 in the SDSUPPORT branch, ~line 2798)
  — motion planner blocks; smoother SD printing. LPC1768 has plenty of
  RAM for this

## Build & validate

```shell
pio run -e LPC1768
```

## Flashing (SD card, no host needed)

The SKR 1.3 (LPC1768) flashes itself from the onboard SD slot:

1. Copy `.pio\build\LPC1768\firmware.bin` to the **root of a small
   FAT32 SD card** (4-8 GB is most reliable; the LPC1768 bootloader
   is picky about cards). Ideally the card contains only
   `firmware.bin`.
2. Power off the printer, insert the card into the **onboard SD slot**
   (the same slot used for printing).
3. Power on. The bootloader flashes the firmware and **renames the
   file to `FIRMWARE.CUR`** when done - that rename is the success
   indicator. Takes ~10-30 s; the LCD may stay dark during this.
4. Power off, remove the card (or delete `firmware.bin`), insert the
   printing SD card, power on. First boot auto-initializes EEPROM
   (`EEPROM_INIT_NOW`) and the display shows "Ender-3".
5. If the flash fails (no `.CUR` rename, blank screen): try a
   different/smaller SD card. Do not interrupt power during the flash.

Note: every re-flash wipes tuned EEPROM values (MPC model, bed PID,
probe offset, E-steps) because of `EEPROM_INIT_NOW` - comment it out
once tuning is done.

## After flashing (operational)

1. EEPROM: `EEPROM_INIT_NOW` (in the plan) auto-initializes EEPROM on
   the first boot after each flash — no manual step needed. Only if it
   was disabled or EEPROM seems corrupted: `M502` then `M500`.
   NOTE: re-flashing the same build wipes tuned EEPROM values back to
   defaults; comment out `EEPROM_INIT_NOW` once tuning is done and
   settings should survive re-flashes.
2. MPC autotune hotend — from the LCD: Configuration → Advanced
   Settings → Temperature → MPC Autotune (runs `M306 T`; heats to
   ~215 °C at bed center and measures the model). Watch progress on
   the status screen, then `M500` to save.
3. Bed PID autotune — from the LCD: Configuration → Advanced Settings →
   Temperature → Bed PID Autotune (runs `M303 E-1`; ~60 °C for PLA).
   Watch progress on the status screen, then `M500` to save.
4. Probe Z-offset calibration — from the LCD: Configuration →
   Advanced Settings → Probe Offsets → Probe Z Wizard (homes, probes,
   then a paper test at the same spot; sets `M851 Z` automatically),
   then `M500`
5. E-steps calibration (no host needed, all from the LCD + SD card):
   - Heat the hotend to printing temp (Temperature menu → set Nozzle
     to ~215 °C; preheat presets are disabled)
   - Use the LCD to extrude slowly until filament flows, then mark the
     filament ~120 mm above the extruder inlet (e.g. with a ruler)
   - Extrude 100 mm from the LCD: Prepare → Move Axis → Extruder
     (move in 10 mm steps ×10; the extruder only moves when hot)
   - Measure the remaining distance from the mark to the inlet
     (`used = 120 − remaining`)
   - New E-steps = current E-steps × 100 / used
     (current value: 830 from the plan — double the old 1.8° value of 415
     for the 0.9° LDO motor; check with `M503` or the
     Configuration → Advanced Settings → Steps/mm menu)
   - Set it from the LCD: Configuration → Advanced Settings →
     Steps/mm → E0 (if the Steps/mm menu is enabled), or put
     `M92 E<new>` + `M500` in a small `.gcode` file on the SD card and
     "print" it
   - Repeat once to confirm, then `M500`
6. Verify axis directions (LCD: Prepare → Move Axis; test with the
   nozzle well above the bed, e.g. after homing Z sits at 10 mm thanks
   to `Z_AFTER_HOMING`):
   - X: jog +10 mm → nozzle must move **right** (toward X=220);
     if left, flip `INVERT_X_DIR`
   - Y: jog +10 mm → bed must move toward the **back** (nozzle ends up
     relatively at the front); if the bed comes toward you, flip
     `INVERT_Y_DIR`
   - Z: jog +10 mm → gantry must move **up**; if down, flip
     `INVERT_Z_DIR`
   - E: jog +10 mm (hotend at printing temp only) → filament must be
     **pushed out**; if it retracts, flip `INVERT_E0_DIR`
   - If any axis is wrong: re-flash with the corresponding
     `INVERT_*_DIR` toggled in Configuration.h (~lines 1903-1914).
     (A runtime `M569` fix is possible but re-flashing is cleaner.)
   - The plan sets INVERT_X/Y true, Z/E false (matching the proven
     2022 config) — verify anyway before the first homing
7. Junction deviation: set at runtime with `M205 J<value>` (e.g. in slicer
   start G-code). Preferred value: `M205 J0.1` — proven at 40 mm/s print
   speeds; revisit if speeds increase. Can also be saved with `M500`.
   Tuning note: no autotune exists — print a test cube with sharp
   corners at your real print speed and look for ghosting/ringing
   trailing behind corners: too much ringing → lower the value;
   visible corner slowdowns but no ringing → raise it. Typical useful
   range 0.01-0.2. Tune acceleration (step 9) first, then this.
8. Linear Advance: set at runtime with `M900 K<value>` (e.g. in slicer
   start G-code). Preferred value: `M900 K0.04` (was 0.05–0.1 before).
   Can also be saved with `M500`.
   Test: print a K-factor calibration pattern (generator:
   <https://marlinfw.org/docs/tools/lin_advance_kfactor.html>) — a
   single-perimeter print where K increases continuously along the
   length. Read it at your real print speed/material: too-low K shows
   bulges at line starts, too-high K shows gaps at line ends; the best
   K is where both ends look clean. The generated `.gcode` goes
   straight on the SD card. K is filament-specific — retest when
   switching materials. Also retest after the 0.9° extruder motor swap:
   doubled E resolution can shift the best K slightly.
9. Motion tuning (all runtime-editable, save with `M500`):
    - `M203 X100 Y100 Z5 E50` — max feedrates (mm/s)
    - Note: at 830 steps/mm, E50 = ~41.5k steps/s — fine for LPC1768, but
      if fast retracts stutter, lower E feedrate (e.g. E25) or drop
      `E0_MICROSTEPS` to 8
    - `M201 X500 Y500 Z100 E5000` — max accelerations (mm/s²)
    - `M204 P500 R500 T500` — print / retract / travel acceleration
    - `M503` — report current values
    - Increase `M204 P` gradually (e.g. 500 → 800 → 1000) and test
      print; back off when ringing/ghosting appears on surfaces
    - LCD: Configuration → Acceleration / Max Feedrate menus also edit
      these (saved with the "Store Settings" menu item)
