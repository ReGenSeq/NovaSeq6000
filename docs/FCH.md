# FCH — Flow Cell Holder

[← back to index](README.md)

**Confidence in name/role:** Medium-high. Response-type namespace is
`Illumina.Novaseq.IX.Fch_2` (the `_2` suffix's meaning — hardware revision,
protocol revision, or something else — is not known). The command set is
squarely about holding, sealing, thermally cycling, and mechanically aligning
a flow cell: vacuum hold-down, engage/disengage, thermal zones, tip/tilt
motors, a door, an RFID reader, and a status lightbar. Unlike FLU/BIM/RCA,
FCH's commands take a `holder`/`zone` argument for `left`/`right` rather than
having a separate left/right board — one FCH serves both flow-cell bays.

## Overview

FCH appears to own the physical bay each flow cell sits in:

- **Thermal**: 6 independently-addressable heater zones per side
  (`fc_{left,right}_{front,mid,back}_top`) plus zone-level temperature
  readback and enable/disable.
- **Tip/tilt**: 3 motors (`cone`, `flat`, `v`) that presumably tilt the
  optical plane of the flow cell into focus/alignment.
- **Flow cell hold**: vacuum-driven hold-down (`engage`/`disengage`/`home`),
  with vacuum pressure readback and a combined state readout.
- **FC door**: open/close a door over the flow-cell bay.
- **RFID**: reads/writes the flow cell's own ID tag (`rfid: fc_0`).
- **Lightbar**: an RGB status indicator per side, presumably showing the
  operator load/ready/error state of that bay.
- Plus the shared **common-firmware** commands also seen on other boards —
  see [common firmware commands](README.md#common-firmware-commands-vs-board-specific-commands)
  in the index: `ix_ix_echo_string`, `ix_io_get`, `ix_log_get_list`,
  `ix_log_replay_server`, `ix_rfid_tag_get/set`, `ix_fpga_counters_reset`.

## Communication

See [protocol basics](README.md#protocol-basics) in the index for the
request/response shape and the `IxID` requirement. No FCH endpoint IP was
captured in this repo's material — only FLU's is (`command_example.py`).

## Commands

### Thermal control

#### `ix_thermal_tgt_set`
Sets a heater zone's target temperature.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `zone` | str | `fc_left_front_top`, `fc_left_mid_top`, `fc_left_back_top`, `fc_right_front_top`, `fc_right_mid_top`, `fc_right_back_top` | 6 zones, 3 per side |
| `milli_degrees_C` | int | `20000, 38000, 40000, 55000, 60000` | target temp in milli-°C (i.e. 20–60 °C); discrete setpoints seen, likely correspond to specific sequencing chemistry steps rather than a continuous range |

**Response** `ix_thermal_no_rsp` — `Exception`, `Message` only.

**Example**
```json
// Tx
{"IxType":"ix_thermal_tgt_set","zone":"fc_left_back_top","milli_degrees_C":20000,"ResponseType":"Illumina.Novaseq.IX.Fch_2.ix_thermal_no_rsp, Illumina.Novaseq.IX.Fch_2, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":12549}
// Rx
{"IxType":"ix_thermal_no_rsp","IxID":12549,"Exception":"ix_exception_id_success","Message":""}
```

#### `ix_thermal_enable`
Enables or disables active temperature control for a zone.

**Args**: `zone` (same 6 values), `enable_nDisable` (bool).
**Response** `ix_thermal_no_rsp` — `Exception`, `Message` only.

#### `ix_thermal_temperature_get`
Requests the current (measured) temperature for a zone.

**Args**: `zone` (only right-side zones observed in this run:
`fc_right_front_top`, `fc_right_mid_top`, `fc_right_back_top`),
`calibrated_nRaw` (bool — `True` seen; presumably `False` returns raw/uncalibrated ADC).
**Response** `ix_thermal_temperature_rsp`: `milli_degrees_C` (int, min 19491
max 61964 — roughly 19.5–62 °C), `Exception`, `Message`.

### Flow cell tip/tilt alignment

#### `ix_tip_tilt_move`
Commands one alignment motor to an absolute position.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `motor` | str | `cone`, `flat`, `v` | 3-point tilt actuation |
| `position_nm` | int | min 0, max 5846250 | target position, nanometers |
| `max_retries` | int | `3` | |

**Response** `ix_tip_tilt_motion_rsp` (see below).

#### `ix_tip_tilt_get_position`
Requests a motor's current position. **Args**: `motor` (`cone`/`flat`/`v`).
**Response**: `ix_tip_tilt_motion_rsp`.

#### `ix_tip_tilt_motion` (response payload, shared by both commands above)

| field | type | observed | notes |
|---|---|---|---|
| `motor` | str | `cone`, `flat`, `v` | echoes which motor |
| `position_nm` | int | min -7500, max 5852500 | resulting/current position |
| `first_error_nm` | int | min -10000, max 8750 | error before any retry |
| `final_error_nm` | int | min -6250, max 6250 | error after retries settle |
| `move_attempts` | int | `0, 1, 2` | how many retries it took |
| `Exception` / `Message` | str | success / `''` | |

### Flow cell hold (vacuum seal)

#### `ix_flowcell_hold_home`
Homes the hold mechanism for one side.
**Args**: `holder` (`left`/`right`). **Response**: `ix_flowcell_hold_state_rsp`.

#### `ix_flowcell_hold_engage`
Engages or disengages the vacuum hold for one side.
**Args**: `holder` (`left`/`right`), `engage_nDisengage` (bool).
**Response**: `ix_flowcell_hold_state_rsp`.

**Example**
```json
// Tx
{"IxType":"ix_flowcell_hold_home","holder":"left","ResponseType":"Illumina.Novaseq.IX.Fch_2.ix_flowcell_hold_state_rsp, Illumina.Novaseq.IX.Fch_2, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":1049}
// Rx
{"IxType":"ix_flowcell_hold_state_rsp","IxID":1049,"state":"ix_flowcell_hold_state_home","y_state":"ix_flowcell_hold_state_home","clamp_state":"ix_flowcell_hold_state_home","vacuum_sealed":false,"Exception":"ix_exception_id_success","Message":""}
```

#### `ix_flowcell_hold_state` (response payload, shared across the hold commands)

| field | type | observed | notes |
|---|---|---|---|
| `state` | str | `ix_flowcell_hold_state_engaged`, `_disengaged`, `_home` | overall hold state |
| `y_state` | str | `ix_flowcell_hold_state_home` | a secondary axis' state |
| `clamp_state` | str | `ix_flowcell_hold_state_home` | mechanical clamp state |
| `vacuum_sealed` | bool | `True`, `False` | whether vacuum seal is currently holding |
| `Exception` | str | `ix_exception_id_success`, `ix_exception_id_errno_error`, `ix_exception_id_errno_state_trans` | the only command in this board observed returning non-success codes — e.g. requesting an invalid transition |
| `Message` | str | `''`, `'Error'`, `'Invalid state transition'` | |

#### `ix_flowcell_hold_vacuum_pressure_get` / `ix_flowcell_hold_vacuum_pressure`
Reads the vacuum pressure holding the flow cell down.
**Args**: `holder` (`left`/`right`).
**Response** `ix_flowcell_hold_vacuum_pressure_rsp`: `pressure_kPa` (int, min
-87 max -78 — negative = vacuum), `Exception`, `Message`.

#### `ix_flowcell_hold_enable` / `ix_flowcell_hold_empty`
Enables/disables the hold subsystem as a whole. **Args**:
`enable_nDisable` (bool — only `False` observed). **Response**
`ix_flowcell_hold_empty_rsp` — `Exception`, `Message` only.

### Flow cell door

#### `ix_fc_door_position_set` / `ix_fc_door_position`
Opens/closes, or reads, the door over the flow-cell bay.
**Args** (set only): `position` (`ix_fc_door_pos_open`,
`ix_fc_door_pos_closed`). **Response** `ix_fc_door_position_rsp`: `position`
(same enum, get only), `Exception`, `Message`.

**Example**
```json
// Tx
{"IxType":"ix_fc_door_position_set","position":"ix_fc_door_pos_open","ResponseType":"Illumina.Novaseq.IX.Fch_2.ix_fc_door_position_rsp, Illumina.Novaseq.IX.Fch_2, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null","IxID":1083}
```

### Status lightbar

#### `ix_lightbar_set`
Sets an RGB status light for one side, solid or blinking.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `lightbar` | str | `left`, `right` | |
| `red` / `grn` / `blu` | int | `0, 66, 84, 85, 110` | 0–255-ish color channels; only a couple of colors seen (amber-ish `110/85/0`, green `0/66/0`) |
| `state` | str | `ix_lightbar_on`, `ix_lightbar_blink` | solid vs. blinking |

**Response** `ix_lightbar_rsp` — `Exception`, `Message` only.

**Example** — blinking amber while busy, then solid green when done:
```json
{"IxType":"ix_lightbar_set","lightbar":"right","red":110,"grn":85,"blu":0,"state":"ix_lightbar_blink","IxID":1696}
{"IxType":"ix_lightbar_set","lightbar":"right","red":0,"grn":66,"blu":0,"state":"ix_lightbar_on","IxID":1733}
```
(`ResponseType`/full envelope omitted above for brevity — same shape as other examples.)

### Flow cell RFID

#### `ix_rfid_tag_get` / `ix_rfid_tag_set`
Reads/writes the flow cell's RFID tag. **Args**: `rfid` (`fc_0` — only one FC
slot's tag seen; presumably a second value exists for a two-flow-cell
config). **Response** `ix_rfid_tag_get_rsp` / `ix_rfid_tag_set_rsp`: `error`
(`ix_rfid_error_status_noerr`), `Exception`, `Message`. Note: this is a
**common-firmware** command (`Illumina.Voyager.IX.IxDevices.CommonCommands.Firmware.Rfid`,
`Illumina.NovaSeq.IX, Version=1.7.5.27`) — CHM has the same command for buffer
bottle tags.

### Sensors and monitoring

#### `ix_oas_pressure_get`
**Args**: `average` (bool, `True` only). **Response**
`ix_oas_pressure_get_rsp`: `value` (int; `-950, 6750, 6850` — likely an
optics/air pressure sensor given "oas"; unit not established), `Exception`,
`Message`.

#### `ix_fan_get_tach`
**Args**: `fan` (`opt_fan1`, `opt_fan2` — "optics" fans, cooling the imaging
path near the flow cell). **Response** `ix_fan_get_tach_rsp`: `fan` (echoed),
`pulses_per_minute` (int, ~10000–10500), `Exception`, `Message`.

#### `ix_ilmn_pwr_voltage_monitor_get` / `ix_ilmn_pwr_current_monitor_get`
**Args**: `pwr_name` — voltage variant saw
`12v_pwr, 24v_pwr, 48v_pwr, 24v_mib_pwr, 12v_mib_pwr` ("mib" rail is unique
to FCH among the devices seen so far); current variant saw
`12v_pwr, 12v_mib_pwr, 24v_mib_pwr`. **Response**: min/max/avg voltage_mV
(12088–48854) or current_mA (6–939), `Exception`, `Message`.

#### `ix_ilmn_pwr` (notification, not a command)
Spontaneous power-rail push, same shape as [FLU's](FLU.md#ix_ilmn_pwr-notification-not-a-command).
`pwr_name` seen: `12v_mib_pwr`.

#### `ix_io_no` (notification, not a command)
Spontaneous I/O pin-state push — see
[Commands vs. notifications](README.md#commands-vs-notifications). Confirmed
directly in the raw logs: `IxID: -1`, no prior `Tx`.

**Args** (the pushed payload)

| field | notes |
|---|---|
| `ix_io_notify.name` | pin name — `door_open`, `door_midway`, `fc_lf_present`, `fc_rf_present`, `fc_lb_present`, `fc_rb_present`, `pressure_left`, `pressure_right`, `valve_left_en`, `valve_right_en`, `t1_home`/`t2_home`/`t3_home` |
| `ix_io_notify.state` | bool |
| `ix_io_notify.input_nOutput` | bool — whether this pin is an input or output on the board |

**Example**
```json
{"IxType":"ix_io_notify","IxID":-1,"name":"valve_right_en","state":true,"input_nOutput":false}
```

### FPGA counters

#### `ix_fpga_counters_reset`
**Args**: `counter` (`ix_fpga_counter_mib_soft_err`, `ix_fpga_counter_fch_soft_err`,
`ix_fpga_counter_mib_link_down`, `ix_fpga_counter_fch_link_down`) — link/error
counters for the FCH's own FPGA link and the "mib" link. **Response**
`ix_fpga_counters_no_rsp` — `Exception`, `Message` only.

### Diagnostics and logging (common firmware)

These are shared across boards — see
[common firmware commands](README.md#common-firmware-commands-vs-board-specific-commands).

- **`ix_ix_echo_string`** — loopback test; `whatever` in, `response` out.
- **`ix_io_get`** — snapshot of all I/O pin states (response fields not
  captured beyond the envelope + `Exception`/`Message` in this run).
- **`ix_log_get_list`** — lists available onboard log streams.
- **`ix_log_replay_server`** — streams a named log back to the host. `name`
  values seen include per-subsystem log channels: `fc_door`, `fch_left`,
  `fch_right`, `tt_flat`/`tt_cone`/`tt_v` (tip/tilt), `thermal`, `rfid`, `io`,
  `chm_server`, `dcmtr_fc_{lf,rf,lb,rb,ly,ry}` (flow-cell door/holder
  motors?), `ads1258_adc0/1/2` and `ads1258_mib_adc0` (ADC channels),
  `max31790` (fan controller chip), `qspi0/1`, `net_tcp`, `ix`, `filesystem`,
  `id_server`, `macaddr`, `bcmd_client`/`bcmd_server`, `esst`, `seq_mngr`,
  `tec_wdg`, `ftp`, `fanservo`, `log_server` — a useful map of FCH's internal
  subsystems even beyond what has an external IX command.

## Open questions

- What does the `Fch_2` namespace suffix mean (hardware/protocol revision)?
- Units and calibration for `ix_oas_pressure_get`'s `value`.
- Whether `rfid: fc_0` has a sibling `fc_1` in configurations with two active
  flow cells, or whether that's handled by `holder`/side elsewhere.
- Full field list for `ix_io_get`'s response (only the envelope + status
  fields were captured — no actual pin snapshot appeared as a distinct field
  in this run's traffic).
- Whether `ix_thermal_temperature_get`/`_temperature`'s left-side zones
  simply weren't exercised in this run, or whether left-side temperature
  readback works differently.
