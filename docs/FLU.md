# FLU — Fluidics board

[← back to index](README.md)

**Confidence in name/role:** High. Response-type namespace
`Illumina.Novaseq.IX.Flu` appears directly in captured traffic, and every
command matches what a reagent-delivery fluidics path needs: pumps, valves,
pressure/flow sensing.

## Overview

FLU drives the fluidics path that pulls reagent from source bottles/cartridges
and pushes it across the flow cell: two syringe pumps, a wash-selector valve,
and a bank of pressure/flow sensors, plus power monitoring for the pump and
degasser motors.

Every command captured in this run carries a `_b` / `_B` suffix
(`wsv_side_b`, `pump_side_b_1`, `pump_side_b_2`, `press1_B` … `press4_B`,
`press_bypass_B`, `press_waste_B`, `flow_B`) — this was a side-B-only
sequencing run. **Open question:** whether side A traffic would appear under
this same `FLU` device name (i.e. one board serves both sides via `_a`/`_b`
args) or under a second, not-yet-seen device name. Nothing in this log
answers that; check a dual-side run's logs before assuming either way.

## Communication

- Example endpoint seen in `../command_example.py`: `169.254.206.140:5555`
  (JSON over TCP). See the [protocol basics](README.md#protocol-basics) in the
  index for the general request/response shape and the `IxID` requirement.

## Commands

### Pumps

#### `ix_pump_aspirate`
Draws reagent into a pump's syringe from the given port.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `pump` | str | `pump_side_b_1`, `pump_side_b_2` | which syringe pump |
| `volume_ul` | int | min 33, max 1000 | draw volume, microliters |
| `rate_ul_per_min` | int | min 200, max 5500 | draw rate |
| `port` | str | `ix_pump_port_input`, `ix_pump_port_bypass` | source port to draw from |

**Response** `ix_pump_empty_rsp` — `Exception`, `Message` only (no data
fields; "empty" response, success/failure only).

**Example**
```json
// Tx
{"IxType":"ix_pump_aspirate","pump":"pump_side_b_1","volume_ul":1000,"rate_ul_per_min":3500,"port":"ix_pump_port_bypass","ResponseType":"Illumina.Novaseq.IX.Flu.ix_pump_empty_rsp, Illumina.Novaseq.IX.Flu, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":18149}
```

#### `ix_pump_dispense`
Pushes reagent out of a pump's syringe to the given port.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `pump` | str | `pump_side_b_1`, `pump_side_b_2` | which syringe pump |
| `volume_ul` | int | min 41, max 1000 | dispense volume, microliters |
| `rate_ul_per_min` | int | min 100, max 6000 | dispense rate |
| `port` | str | `ix_pump_port_input`, `ix_pump_port_bypass` | destination port |

**Response** `ix_pump_empty_rsp` — `Exception`, `Message` only.

**Example**
```json
// Tx
{"IxType":"ix_pump_dispense","pump":"pump_side_b_1","volume_ul":527,"rate_ul_per_min":250,"port":"ix_pump_port_bypass","ResponseType":"Illumina.Novaseq.IX.Flu.ix_pump_empty_rsp, Illumina.Novaseq.IX.Flu, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":26634}
// Rx
{"IxType":"ix_pump_empty_rsp","IxID":18014,"Exception":"ix_exception_id_success","Message":""}
```
(Rx shown is from a paired `ix_pump_dispense_all` call — the response shape
is identical for all three dispense/aspirate commands.)

#### `ix_pump_dispense_all`
Fully empties a pump's syringe (dispense to completion, no explicit volume)
to the given port.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `pump` | str | `pump_side_b_1`, `pump_side_b_2` | |
| `rate_ul_per_min` | int | `250, 2500, 5000, 10000, 15000` | dispense rate — note this jumps in large discrete steps, unlike the continuous ranges on `ix_pump_dispense`/`ix_pump_aspirate` |
| `port` | str | `ix_pump_port_input`, `ix_pump_port_output`, `ix_pump_port_bypass` | destination — the only command that showed the `output` port option |

**Response** `ix_pump_empty_rsp` — `Exception`, `Message` only.

#### `ix_pump_set_port`
Switches a pump's active port without moving fluid.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `pump` | str | `pump_side_b_1`, `pump_side_b_2` | |
| `port` | str | `ix_pump_port_input` | only `input` seen; other port enum values presumably valid too |

**Response** `ix_pump_empty_rsp` — `Exception`, `Message` only.

### Valves

#### `ix_valve_set_position`
Sets the wash-selector valve to one of its positions.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `valve` | str | `wsv_side_b` | wash-selector valve, side B |
| `position` | int | `0`, `1` | binary position; which physical flow path each value selects is not decoded from this data alone |

**Response** `ix_valve_set_position_rsp`

| field | type | observed | notes |
|---|---|---|---|
| `expected_counts` | int | 0 | always 0 in this run |
| `actual_counts` | int | 0 | motor/encoder counts for the move, always 0 here |
| `retry_count` | int | 0 | |
| `Exception` | str | `ix_exception_id_success` | |
| `Message` | str | `''` | |

**Example**
```json
// Tx
{"IxType":"ix_valve_set_position","valve":"wsv_side_b","position":0,"ResponseType":"Illumina.Novaseq.IX.Flu.ix_valve_set_position_rsp, Illumina.Novaseq.IX.Flu, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":18006}
// Rx
{"IxType":"ix_valve_set_position_rsp","IxID":18006,"expected_counts":0,"actual_counts":0,"retry_count":0,"Exception":"ix_exception_id_success","Message":""}
```

### Sensor logging

FLU exposes a small time-series logger for its pressure/flow sensors —
start/stop recording, then pull stats or raw counts back.

#### `ix_flu_logger_start` / `ix_flu_logger_stop`
Starts or stops recording readings for one sensor channel.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `sensor` | str | `press1_B`, `press2_B`, `press3_B`, `press4_B`, `press_bypass_B`, `press_waste_B`, `flow_B` | which sensor channel |

**Response** `ix_flu_logger_no_rsp` — `Exception`, `Message` only.

#### `ix_flu_logger_getstats`
Returns summary statistics over a recorded window for one sensor.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `sensor` | str | `press1_B`…`press4_B`, `press_bypass_B`, `press_waste_B` | (not `flow_B` in this run) |
| `startrec` | int | `0` | record index to start from |
| `endrec` | int | `0` | record index to end at — `0`/`0` here likely means "whole buffer" |

**Response** `ix_flu_logger_getstats_rsp`

| field | type | observed | notes |
|---|---|---|---|
| `count` | int | min 6, max 1572 | number of samples in the window |
| `timestamp_msec` | int | min 353821200, max 506840511 | free-running device timestamp, not wall clock |
| `read_min` / `read_max` / `read_mean` | int | ~800–2700 | raw sensor units, not yet mapped to physical pressure |
| `read_stddev` | int | min 0, max 289 | |
| `Exception` / `Message` | str | success / `''` | |

#### `ix_flu_logger_get_numrecs`
Returns how many records are currently buffered for a sensor.

**Args**: `sensor` — only `flow_B` seen.

**Response** `ix_flu_logger_get_numrec_rsp`: `numrecs` (int, min 14 max 512),
`Exception`, `Message`.

### Power monitoring

#### `ix_ilmn_pwr_voltage_monitor_get` / `ix_ilmn_pwr_current_monitor_get`
Requests a voltage or current summary for a named power rail.

**Args**: `pwr_name` — voltage variant saw
`pump1_pwr, pump2_pwr, pump3_pwr, pump4_pwr, degasser_pwr, 12v_pwr, 24v_pwr`;
current variant saw `pump1_pwr, pump2_pwr, pump3_pwr, pump4_pwr`.

**Response** `ix_ilmn_pwr_voltage_monitor_rsp` / `ix_ilmn_pwr_current_monitor_rsp`:
`min_/max_/avg_voltage_mV` (min 12016 – max 24196) or
`min_/max_/avg_current_mA` (min 20 – max 648), plus `Exception`, `Message`.

#### `ix_ilmn_pwr` (notification, not a command)
A spontaneous, unsolicited push from the board — not something you send. See
[Commands vs. notifications](README.md#commands-vs-notifications) in the
index. Carries one power-rail sample:

| field | notes |
|---|---|
| `ix_ilmn_pwr_mon.pwr_name` | `12v_pwr` in this run |
| `ix_ilmn_pwr_mon.voltage_nCurrent` | `True` → this sample is a voltage reading |
| `ix_ilmn_pwr_mon.in_range` | whether the reading is within the rail's expected band |
| `ix_ilmn_pwr_mon.voltage_mV` | the sample itself, e.g. `12072`–`12120` |

### Diagnostics

#### `ix_ix_echo_string`
Common-firmware loopback test — sends a string, expects it echoed back.
Args: `whatever` (str, `"Echo string"` in this run). Response
`ix_ix_echo_string_rsp`: `response` (echoes `whatever`), `Exception`, `Message`.

## Open questions

- What do `wsv_side_b` valve `position` values `0`/`1` physically select?
- Is there a side-A equivalent under this same device name, or a second board?
- What are `read_min`/`read_max`/`read_mean` in `ix_flu_logger_getstats`
  measured in — raw ADC counts, or already-scaled pressure units?
- `ix_pump_port_output` only appeared on `ix_pump_dispense_all` — is it valid
  for the other pump commands too, or specific to a full-empty operation?
