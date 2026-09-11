# SYST — System controller

[← back to index](README.md)

**Confidence in name/role:** High. Response-type namespace
`Illumina.Novaseq.IX.Syst` is confirmed. The command set is chassis-wide,
not tied to any single subsystem: pump current across the whole instrument
(not one board's pumps), a refrigeration compressor, a bank of named fans
spanning multiple subsystems (`fch_fan*`, `rca_fan*`, `comp_fan`, `ee_fan`,
`fe_fan`), and the main 12V/24V power-rail monitoring. Reads as the
instrument's overall electrical/environmental housekeeping controller,
separate from any single flow-cell-side or reagent-handling board.

## Overview

SYST doesn't own a physical mechanism the way FLU/FCH/RCA/BIM/CIB/FIB do —
its commands read current/voltage/tachometer/level sensors that describe the
instrument as a whole:

- **Pump current** (`ix_syst_pump_current_get`) — by `zone`
  (`pumpopt`, `pumpfch`, `pumprca`), i.e. the vacuum/optics pump, the FCH
  pump, and the RCA pump, monitored centrally rather than by each board.
- **Compressor current** — the instrument's refrigeration compressor
  (feeding the RCA chillers and/or FCH cooling).
- **Fan tachometers** — a wide set of named fans across subsystems:
  `ee_fan`, `fe_fan` (electronics enclosure / front-end?), `comp_fan`
  (compressor), `fch_fan1`-`3`, `rca_fan1`-`3`.
- **A liquid-level sensor** (`level0_ch1`, via `ix_io_no`) — likely a waste
  or coolant reservoir level switch.
- **12V/24V power rail monitoring**, same pattern as every other board.

## Communication

See [protocol basics](README.md#protocol-basics) in the index. No SYST
endpoint IP was captured in this repo's material.

## Commands

### Pump and compressor current

#### `ix_syst_pump_current_get`
Reads instrument-level pump motor current for one pump zone.

**Args**: `zone` (`pumpopt`, `pumpfch`, `pumprca`).
**Response** `ix_syst_pump_current_rsp`: `current_milli_amps` (int, min 714
max 1486), `Exception`, `Message`.

**Example**
```json
{"IxType":"ix_syst_pump_current_get","zone":"pumpfch","IxID":17589}
{"IxType":"ix_syst_pump_current_get","zone":"pumpopt","IxID":17596}
```

#### `ix_syst_compressor_current_get`
Reads the refrigeration compressor's motor current. No args.
**Response** `ix_syst_compressor_current_get_rsp`: `current_milli_amps`
(int, min 40 max 2808 — the wide range likely reflects compressor
start-up inrush current vs. steady-state), `Exception`, `Message`.

### Fans

#### `ix_fan_get_tach`
Reads a fan's tachometer.

**Args**: `fan` — `rca_fan1`, `rca_fan2`, `rca_fan3`, `fch_fan1`,
`fch_fan2`, `fch_fan3`, `comp_fan`, `ee_fan`, `fe_fan`.
**Response** `ix_fan_get_tach_rsp`: `fan` (echoed), `pulses_per_minute`
(int, min 3640 max 8936), `Exception`, `Message`.

### Power rail monitoring

#### `ix_ilmn_pwr_voltage_monitor_get` / `ix_ilmn_pwr_current_monitor_get`
**Args**: `pwr_name` (`12v_pwr`, `24v_pwr`). **Response**: min/max/avg
`voltage_mV` (12224-24313) or `current_mA` (164-10128), `Exception`,
`Message`.

### Liquid level (notification)

#### `ix_io_no`
Spontaneous I/O pin-state push — see
[Commands vs. notifications](README.md#commands-vs-notifications).

**Args** (pushed payload): `ix_io_notify.name` — `level0_ch1` (the only
name seen for SYST; a level-switch channel, toggling `True`/`False` — likely
a waste or coolant reservoir sensor tripping as fluid crosses a threshold);
`state` (bool); `input_nOutput` (`True`).

**Example**
```json
{"IxType":"ix_io_notify","IxID":-1,"name":"level0_ch1","state":true,"input_nOutput":true}
```

#### `ix_ilmn_pwr` (notification, not a command)
Spontaneous power-rail push, same shape as other boards'. `pwr_name` seen:
`12v_pwr`, `24v_pwr`.

### Diagnostics (common firmware)

- **`ix_ix_echo_string`**, **`ix_io_get`** — same as every other board.

## Open questions

- What `level0_ch1` actually measures (waste tank, coolant reservoir, drip
  tray — any are plausible from the name alone).
- Whether `pumpopt`/`pumpfch`/`pumprca` current readings duplicate or
  supplement per-board power monitoring (e.g. FCH's own `ix_ilmn_pwr_*`
  rails) — i.e. is SYST a second, independent monitoring path for the same
  physical pumps, or a distinct set of pumps SYST alone controls/monitors.
- Whether `ee_fan`/`fe_fan` stand for "electronics enclosure"/"front end" —
  a guess from the abbreviation, not confirmed.
