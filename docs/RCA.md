# RCALeft / RCARight — Reagent Chiller Assembly

[← back to index](README.md)

**Confidence in name/role:** Medium-high. Response-type namespace
`Illumina.Novaseq.IX.Rca` is confirmed on both boards. The command set —
a temperature-controlled `chiller0` zone, a rotary reagent selector valve,
cartridge presence sensors, sipper up/down, and load/unload/wash/mix/pierce
automation ops — is squarely a reagent-cartridge handling and chilling
station. "RCA" reads naturally as **Reagent Chiller Assembly**, and unlike
BIM/FCH's left/right split (which mirrors the two flow-cell sides), RCA's
two boards look like two independent cartridge bays in the reagent
compartment rather than a per-flowcell-side pair — see
[Left vs. Right, below](#left-vs-right-two-different-cartridge-bays-not-two-flow-cell-sides).

## Overview

Both boards expose the same command set, but the *data* each one produced in
this run points to two different jobs:

- **RCARight** saw the full range: a 24-position `reagent` valve plus a
  6-position `clsv` valve, and every `ix_fluidics_automation_do` op —
  `load`, `unload`, `pierce`, `mix_up`, `mix_down`, `wash`. That's the
  behavior of the main SBS reagent cartridge: pierce its foil seals, mix
  reagents, route them through a many-position selector valve, wash the
  path between uses.
- **RCALeft** only exercised `load`/`unload` and a single static valve
  position (`reagent`, position `6`). Consistent with a simpler cartridge —
  e.g. a buffer or cluster-reagent cartridge that doesn't need piercing/mixing/
  washing in this run — but this run may simply not have exercised its full
  range either; treat RCALeft's smaller footprint as "not observed" rather
  than "doesn't support."

Both boards see the same cartridge-bay I/O notification names
(`cart_a1`, `cart_a2`, `tray_seq`, `tray_wash`, `door1_closed`,
`sipper_up`/`sipper_down`) and the same RFID tag set (`sma`, `pe`, `sbs` —
read as reagent cartridge type/lot tags), so each bay can apparently identify
and handle more than one cartridge type; which cartridge a given run puts in
which bay isn't determined by this data.

### "Left vs. Right": two different cartridge bays, not two flow-cell sides

Nothing in the captured commands ties `RCALeft`/`RCARight` to flow-cell side
A/B the way FLU's `_side_b` suffixes do — RCA's arguments never mention a
side. Take the "Left/Right" naming as physical-bay position in the reagent
compartment until confirmed otherwise.

## Communication

See [protocol basics](README.md#protocol-basics) in the index. No RCA
endpoint IP was captured in this repo's material.

## Commands

### Reagent cartridge chiller (thermal)

#### `ix_thermal_tgt_set` / `ix_thermal_enable`
Sets the target temperature / enables active control for the chiller zone.

**Args**: `zone` (only `chiller0` seen — one chiller zone per board),
`milli_degrees_C` (`7000` only, i.e. 7 °C — a refrigeration setpoint, well
below FCH's 20-60 °C flow-cell zones), `enable_nDisable` (bool, enable only
).
**Response** `ix_thermal_no_rsp` — `Exception`, `Message` only.

#### `ix_thermal_temperature` (RCARight only, in this run)
Reads back the chiller's actual temperature. **Response**
`ix_thermal_temperature_rsp`: `milli_degrees_C` (`7119`, i.e. 7.1 °C —
tracking the 7 °C setpoint closely), `Exception`, `Message`.

### Reagent selector valve

#### `ix_valve_set_position`
Rotates a selector valve to a numbered port.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `valve` | str | `reagent` (both boards), `clsv` (RCARight only) | `clsv` = likely "cluster selector valve" for the exclusion-amp/cluster reagent path |
| `position` | int | RCALeft: `6` only. RCARight: 1-24 (`reagent`) | `reagent` valve has (at least) 24 positions — matches a 24-reagent-position sequencing cartridge |

**Response** `ix_valve_set_position_rsp`

| field | type | observed | notes |
|---|---|---|---|
| `expected_counts` | int | RCALeft: 0. RCARight: 0-3672 | motor step counts expected for the move |
| `actual_counts` | int | RCALeft: 0. RCARight: 0-3711 | counts actually taken |
| `retry_count` | int | `0` | |
| `Exception` / `Message` | str | success / `''` | |

**Example**
```json
// Tx
{"IxType":"ix_valve_set_position","valve":"reagent","position":6,"ResponseType":"Illumina.Novaseq.IX.Rca.ix_valve_set_position_rsp, Illumina.Novaseq.IX.Rca, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":1009}
```

#### `ix_valve_config` (RCARight only, in this run)
Configures a valve's retry behavior and (per the raw log — see note below)
its set of legal positions.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `valve` | str | `reagent`, `clsv` | |
| `avoid_position` | int | `11` (`clsv`), `16` (`reagent`) | a position to skip/avoid when homing or seeking, e.g. a known-bad or seal position |
| `valid_positions` | list[int] | `clsv`: `[23,22,1,4,13,20]` (6 positions). `reagent`: `[1..24]` (24 positions) | **not shown in `RCARight_library.md`** — `parse_logs.py` only records scalar argument values, so this list-valued field was silently dropped when the library file was generated. Confirmed directly from `logs/`; worth patching the parser if you want list args captured going forward. |
| `retry_count` | int | `3` (`clsv`), `4` (`reagent`) | |

**Response** `ix_valve_empty_rsp` — `Exception`, `Message` only.

**Example**
```json
{"IxType":"ix_valve_config","valve":"reagent","avoid_position":16,"valid_positions":[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24],"retry_count":4,"IxID":955349}
```

### Cartridge automation (load / mix / pierce / wash)

#### `ix_fluidics_automation_do`
Drives the cartridge-handling mechanism through one automation step.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `rcam` | str | `rcam` | always this literal value — likely names the single automation mechanism on the board rather than selecting between several |
| `op` | str | RCALeft: `ix_fluidics_automation_op_load`, `_unload`. RCARight: also `_pierce`, `_mix_up`, `_mix_down`, `_wash` | pierce the cartridge's foil seal, mix its contents up/down, wash the fluid path, in addition to loading/unloading |

**Response** `ix_fluidics_automation_no_rsp` — `Exception`, `Message` only.

**Example**
```json
{"IxType":"ix_fluidics_automation_do","rcam":"rcam","op":"ix_fluidics_automation_op_pierce","IxID":17820}
{"IxType":"ix_fluidics_automation_do","rcam":"rcam","op":"ix_fluidics_automation_op_mix_up","IxID":17916}
{"IxType":"ix_fluidics_automation_do","rcam":"rcam","op":"ix_fluidics_automation_op_wash","IxID":956177}
```

#### `ix_fluidics_automation_get_pos`
Reads the cartridge mechanism's current high-level position/state.

**Args**: `rcam` (`rcam`). **Response**
`ix_fluidics_automation_get_pos_rsp`: `pos` — RCALeft:
`ix_fluidics_automation_pos_loaded`, `_unload`. RCARight: also `_wash`,
`_sequence`, `_mix` — plus `Exception`, `Message`.

#### `ix_fluidics_automation_get_motor_step_pos`
Reads a raw motor step count for one automation motor.

**Args**: `rcam` (`rcam`), `motor` (`ix_motor_sipper_z`, `ix_motor_tray_y` —
a vertical sipper axis and a horizontal tray axis).
**Response** `ix_fluidics_automation_get_motor_step_pos_rsp`: `step_pos`
(int, RCALeft min -1505000 max 28945000; RCARight min -1505000 max
104530000 — RCARight's tray/sipper travel is far larger, consistent with the
richer automation it performs), `Exception`, `Message`.

### Fan

#### `ix_fan_set_pwm`
Sets the chiller's cooling fan speed. **Args**: `fan` (`fan_sma`),
`pwm_percentage` (`0, 25, 50`). **Response** `ix_fan_no_rsp` — `Exception`,
`Message` only.

### Cartridge RFID

#### `ix_rfid_tag_get` / `ix_rfid_tag_set`
Reads/writes a cartridge's RFID tag. Common-firmware command, shared with
FCH/CHM — see [FCH's RFID section](FCH.md#flow-cell-rfid).

**Args**: `rfid` (`sma`, `pe`, `sbs` — read as cartridge-type identifiers:
**s**equencing-**b**y-**s**ynthesis reagents, **p**aired-**e**nd mix, and
`sma` unclear — possibly a small-volume/sample-additive cartridge).
**Response**: `error` — `ix_rfid_error_status_noerr` normally, but also
`ix_rfid_error_status_failed_inventory` paired with
`Exception: ix_exception_id_generic_failure` and
`Message: "An error has occurred while executing the command"` — this is the
only device where a non-firmware-common command showed a real failure mode
in the captured data (an empty bay, or a tag that failed to read).

### Diagnostics and logging (common firmware)

Shared across boards — see
[common firmware commands](README.md#common-firmware-commands-vs-board-specific-commands):
`ix_ix_echo_string`, `ix_io_get`, `ix_log_get_list`, `ix_log_replay_server`.
`ix_log_replay_server`'s `name` channel list for RCA: `valve_test`,
`valves`, `thermal`, `stepper`, `clsv`, `reagent`, `chiller0`, `chiller`,
`fansma`, `rfid`, `tec_wdg`, plus the generic channels also seen on FCH
(`net_tcp`, `ix`, `filesystem`, `seq_mngr`, `log_server`, `io`, `spi0`,
`max31790`, `ads1258_adc1`) — the board-specific names here (`clsv`,
`reagent`, `chiller`, `stepper`) line up with the commands above.

### Notification

#### `ix_io_no`
Spontaneous I/O pin-state push — see
[Commands vs. notifications](README.md#commands-vs-notifications). Same
payload shape as FCH's.

**Args** (pushed payload): `ix_io_notify.name` — `tray_seq`, `tray_wash`,
`door1_closed`, `sipper_up`, `sipper_down`, `cart_a1`, `cart_a2` (cartridge-A
present, slots 1/2 — suggests each bay can hold/sense up to 2 cartridges, or
"a1"/"a2" name two sensor positions on one cartridge); `state` (bool);
`input_nOutput` (bool, `True` for all seen — these are all sensor inputs).

#### `ix_ilmn_pwr`
Spontaneous power-rail push, same shape as FLU's/FCH's. `pwr_name` seen:
`24v_pwr`.

## Open questions

- Confirm RCALeft vs. RCARight's actual physical role (buffer/cluster
  cartridge bay vs. reagent cartridge bay) — inferred here from command
  richness, not stated anywhere in the source material.
- What `sma` stands for among the RFID tag types.
- Whether `cart_a1`/`cart_a2` means two sensed positions on one cartridge or
  two separate cartridge slots per bay.
- `parse_logs.py` drops list-valued args (see `ix_valve_config` above) —
  worth checking whether any other command in any device has a hidden
  list-valued field the library `.md`s are silently missing.
