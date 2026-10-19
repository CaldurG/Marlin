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

- `DEFAULT_AXIS_STEPS_PER_UNIT { 80, 80, 400, 415 }` (E: 415 for Titan direct
  drive; old config had 430 — calibrate after flashing)
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
- `Z_MIN_PROBE_ENDSTOP_HIT_STATE LOW` (2.1.x replacement for old
  `Z_MIN_PROBE_ENDSTOP_INVERTING true`)
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

- `AUTO_BED_LEVELING_BILINEAR` (uncomment)
- `GRID_MAX_POINTS_X 2`
- Keep `ENABLE_LEVELING_FADE_HEIGHT` defaults
- `LCD_BED_LEVELING` (uncomment, Configuration.h ~line 2378) — Bed
  Leveling submenu in the LCD: run G29, view/activate the mesh, and
  fade height, all from the display
- `ASSISTED_TRAMMING` (uncomment, Configuration_adv.h ~line 1125) —
  G35 probes the bed corners and reports which screw to turn and how
  much. Set `TRAMMING_POINT_XY { { 35, 30 }, { 190, 30 }, { 190, 205 },
  { 35, 205 } }` (35 mm inset on X, 30 mm on Y, for the 225×235 bed)
  and `TRAMMING_SCREW_THREAD M3_CW` (Ender 3 thumbwheels)

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
- `HEATER_0_MAXTEMP 300` (stock 275; old config 290) — the thermistor
  table (EPCOS 100k, type 1) goes up to 320 °C; 300 leaves headroom
  under the table limit while covering any filament you'd run through
  a stock Ender 3 hotend
- `PID_EDIT_MENU` / `PID_AUTOTUNE_MENU` — DROP (PID no longer used for
  the hotend; M303 does not apply to MPC)
- Preheat 1: `"PLA"` 210 / 55
- Preheat 2: `"PETG"` 240 / 80

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
- `E0_CURRENT 700`, `E0_MICROSTEPS 16`, `E0_RSENSE 0.11`
  (may need a bump for Titan direct drive — increase if it skips)
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

## After flashing (operational)

1. `M502` then `M500` — reset EEPROM to new defaults
2. `M306 T` — MPC autotune hotend (heats to ~215 °C at bed center and
   measures the model; run from LCD via MPC_AUTOTUNE_MENU), then `M500`
3. `M303 E-1 S60` — PID autotune bed (bed stays bang-bang/PID), then `M500`
4. Probe Z-offset calibration (`M851 Z` / paper test), then `M500`
5. E-steps calibration (`M92 E`), then `M500`
6. Verify axis directions; flip `INVERT_*_DIR` if any axis runs backwards
7. Junction deviation: set at runtime with `M205 J<value>` (e.g. in slicer
   start G-code). Preferred value: `M205 J0.1` — proven at 40 mm/s print
   speeds; revisit if speeds increase. Can also be saved with `M500`.
8. E-steps calibration (Titan direct drive, default 415):
   - Heat hotend, mark filament at the extruder inlet
   - `M83` then `G1 E100 F300` — extrude 100 mm slowly
   - Measure actual extruded length, then
     `new_steps = old_steps × 100 / actual_extruded_mm`
   - Apply with `M92 E<new>`, repeat until actual = 100 mm, then `M500`
   - Or set per print via slicer start G-code: `M92 E415`
9. Linear Advance: set at runtime with `M900 K<value>` (e.g. in slicer
   start G-code). Preferred value: `M900 K0.04` (was 0.05–0.1 before).
   Can also be saved with `M500`.
10. Motion tuning (all runtime-editable, save with `M500`):
    - `M203 X100 Y100 Z5 E50` — max feedrates (mm/s)
    - `M201 X500 Y500 Z100 E5000` — max accelerations (mm/s²)
    - `M204 P500 R500 T500` — print / retract / travel acceleration
    - `M503` — report current values
    - Increase `M204 P` gradually (e.g. 500 → 800 → 1000) and test
      print; back off when ringing/ghosting appears on surfaces
    - LCD: Configuration → Acceleration / Max Feedrate menus also edit
      these (saved with the "Store Settings" menu item)
