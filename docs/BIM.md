# BIMLeft / BIMRight — Bulk (Reagent Bottle) Input Module

[← back to index](README.md)

**Confidence in name/role:** Medium. Response-type namespace
`Illumina.Novaseq.IX.Bim` is confirmed on both boards; the full expansion of
"BIM" is not. The command set — a single up/down automation axis, two
"source" bottle-presence/handle sensors, and power monitoring — reads as the
sipper mechanism that lowers into (and raises out of) two bulk reagent
source bottles (wash buffer / SBS wash, most likely), separate from the
cartridge-based reagent handling on [RCA](RCA.md). "Bulk Input Module" is a
descriptive best guess, not a confirmed name.

## Overview

BIMLeft and BIMRight are the simplest boards documented so far: one binary
motion (`up`/`down`), homed against a limit switch, moving a sipper down
into (or up out of) whichever bulk source bottle is in front of it. Two
"source" positions are sensed per board:

- `sour1_present` — bottle 1 detected
- `sour2_present` — bottle 2 detected
- `sour2_handle_down` — bottle 2's handle/cap-lock position

There's no `sour1_handle_down` observed — either bottle 1 doesn't have a
sensed handle, or this run simply never toggled it.

BIMLeft's library only captured the notification/monitoring traffic (no
`ix_fluidics_automation_*` commands appear in `BIMLeft_library.md`); BIMRight
captured the full command set including the up/down moves. Both boards
almost certainly support the same commands — treat BIMLeft's smaller
footprint as "not exercised in this run," matching the same caveat noted for
[RCALeft](RCA.md#overview).

Note BIM's automation command has **no `rcam`-style sub-mechanism argument**
the way RCA's does — `ix_fluidics_automation_do` here takes only `op`
(`up`/`down`), consistent with there being exactly one motion axis on this
board rather than RCA's multi-axis cartridge handler.

## Communication

See [protocol basics](README.md#protocol-basics) in the index. No BIM
endpoint IP was captured in this repo's material.

## Commands

### Sipper motion

#### `ix_fluidics_automation_do`
Moves the sipper up or down.

**Args**: `op` (`ix_fluidics_automation_op_up`, `ix_fluidics_automation_op_down`).
**Response** `ix_fluidics_automation_no_rsp` — `Exception`, `Message` only.

**Example**
```json
// Tx
{"IxType":"ix_fluidics_automation_do","op":"ix_fluidics_automation_op_down","ResponseType":"Illumina.Novaseq.IX.Bim.ix_fluidics_automation_no_rsp, Illumina.Novaseq.IX.Bim, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":17729}
```

#### `ix_fluidics_automation_get_pos`
Reads the sipper's current high-level position.
**Response** `ix_fluidics_automation_get_pos_rsp`: `pos`
(`ix_fluidics_automation_pos_up`, `ix_fluidics_automation_pos_down`),
`Exception`, `Message`.

#### `ix_fluidics_automation_get_motor_step_pos`
Reads the raw motor step count for the sipper axis.
**Response** `ix_fluidics_automation_get_motor_step_pos_rsp`: `step_pos`
(int; `-12000` near the top limit, `183126000` near the bottom — a large
range for what's nominally a two-position axis, suggesting fine step
resolution rather than a coarse mechanism), `Exception`, `Message`.

### Power monitoring

#### `ix_ilmn_pwr_voltage_monitor_get` / `ix_ilmn_pwr_voltage_monitor`
Same pattern as [FLU](FLU.md#power-monitoring). **Args**: `pwr_name`
(`12v_pwr`, `24v_pwr`). **Response**: min/max/avg `voltage_mV`
(12032–24504), `Exception`, `Message`.

#### `ix_ilmn_pwr` (notification, not a command)
Spontaneous power-rail push — see
[Commands vs. notifications](README.md#commands-vs-notifications).
`pwr_name` seen: `12v_pwr`.

### Diagnostics (common firmware)

- **`ix_ix_echo_string`** — loopback test, same as every other board.
- **`ix_io_get`** (BIMRight only, in this run) — I/O snapshot; no data
  fields beyond the envelope were captured.

### Notification

#### `ix_io_no`
Spontaneous I/O pin-state push — see
[Commands vs. notifications](README.md#commands-vs-notifications).

**Args** (pushed payload): `ix_io_notify.name` — `step_limit`, `step_home`
(motor limit/home switches for the up/down axis), `sour1_present`,
`sour2_present`, `sour2_handle_down` (bottle sensors, described above);
`state` (bool); `input_nOutput` (bool, `True` for all seen).

**Example**
```json
{"IxType":"ix_io_notify","IxID":-1,"name":"sour1_present","state":false,"input_nOutput":true}
{"IxType":"ix_io_notify","IxID":-1,"name":"sour2_handle_down","state":false,"input_nOutput":true}
```

## Open questions

- What "BIM" expands to, and what liquid the two source bottles actually
  hold (bulk wash buffer is the likely guess given NovaSeq's known
  consumables, but nothing in this data names it).
- Whether BIMLeft supports the same `ix_fluidics_automation_*` commands as
  BIMRight (near-certain, but not captured in this run's BIMLeft traffic).
- Why only `sour2` has a sensed handle position and not `sour1`.
