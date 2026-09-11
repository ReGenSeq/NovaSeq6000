# FIB — Focus & Illumination Board

[← back to index](README.md)

**Confidence in name/role:** Medium. Response-type namespace is
`Illumina.Novaseq.IX.Fib_2` (same revision-suffix pattern as `Fch_2`/`Cib_2`).
"FIB" most likely expands around **F**ocus / **I**llumination — the command
set covers two clearly distinct subsystems on one board: the excitation
laser sources (red/green, via a "LGM" — laser gain module — sub-namespace)
and a complete autofocus loop (focus laser, Z-axis focus motor, focus
camera, focus scoring). "Fiber Illumination Board" is a plausible
alternative reading if the lasers are fiber-coupled, but nothing here
confirms it either way.

## Overview

Two subsystems, one board:

- **Illumination (`ix_lgm_*`)**: controls the red and green excitation
  lasers used for imaging — power setpoint (closed-loop, "blocking" until
  the power stabilizes within a margin), on/off mode, and a shutter, plus
  status readback (VOA attenuation, diode current, safety interlock state).
- **Autofocus (`ix_focus_*`, `ix_z_*`)**: a separate low-power focus laser
  and small focus camera, a Z-axis motor that moves the objective/optics to
  the computed focus position, and focus-quality feedback (frame capture,
  histogram score, ETF results — "ETF" ties this to
  [CIB's `ix_scan_etf`](CIB.md#ix_scan_etf), which looks like the
  scan-side counterpart of this board's focus-mapping pass).
- **Optics zoom (`ix_zoom_*`)**: a top/bottom zoom position, likely
  switching between two imaging magnifications or channel configurations.
- Plus **common firmware** diagnostics/logging and power monitoring, as on
  every other board.

## Communication

See [protocol basics](README.md#protocol-basics) in the index. No FIB
endpoint IP was captured in this repo's material.

## Commands

### Illumination lasers (LGM)

#### `ix_lgm_mode_set`
Turns a laser on or off. **Args**: `laser` (`ix_lgm_laser_red`,
`ix_lgm_laser_green`), `mode` (`ix_lgm_mode_on`, `ix_lgm_mode_off`).
**Response** `ix_lgm_no_rsp` — `Exception`, `Message` only.

#### `ix_lgm_power_set_blocking`
Sets a laser's output power and blocks until it settles within tolerance.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `laser` | str | `ix_lgm_laser_red`, `ix_lgm_laser_green` | |
| `power_mw` | int | min 50, max 2101 | target optical power, milliwatts |
| `margin_percent` | int | `0, 5, 30` | acceptable tolerance band, percent |
| `margin_delta_mw` | int | `15, 50` | acceptable tolerance band, absolute mW |
| `timeout_ms` | int | `240000, 300000, 600000` | how long to wait for the power to settle (4-10 min) |
| `polling_interval_ms` | int | `200, 10000` | how often power is checked while settling |
| `fitness_span` | int | `3, 4` | number of consecutive good readings required |

**Response** `ix_lgm_no_rsp` — `Exception`, `Message` only.

**Example**
```json
{"IxType":"ix_lgm_power_set_blocking","laser":"ix_lgm_laser_green","power_mw":50,"margin_percent":5,"margin_delta_mw":50,"timeout_ms":240000,"polling_interval_ms":200,"fitness_span":3,"IxID":21722}
{"IxType":"ix_lgm_power_set_blocking","laser":"ix_lgm_laser_red","power_mw":200,"margin_percent":5,"margin_delta_mw":50,"timeout_ms":240000,"polling_interval_ms":200,"fitness_span":3,"IxID":21732}
```

#### `ix_lgm_power_get` / `ix_lgm_power` (response payload)
Reads a laser's current power. **Args** (get): `laser`. **Response**:
`power_mw` (int, min 0 max 2100700 — the max looks like an outlier/error
reading rather than a real 2100 W value; treat very large readings with
suspicion), `Exception`, `Message`.

#### `ix_lgm_status_get` / `ix_lgm_status` (response payload)
Reads laser diagnostic status. **Args** (get): `laser`. **Response**:
`voa_percent` (int, min 15100 max 37700 — variable optical attenuator
setting), `diode_current` (`65500`, `73200`), `interlock_state` (bool,
`True` only in this run — laser safety interlock closed/satisfied),
`Exception`, `Message`.

#### `ix_lgm_shutter_state_set`
Opens/closes the laser shutter. **Args**: `closed` (bool).
**Response** `ix_lgm_no_rsp` — `Exception`, `Message` only.

### Autofocus

#### `ix_z_move` / `ix_z_get_position` / `ix_z_motion` (response payload)
Moves/reads the Z-axis focus motor. **Args** (move): `position_nm` (int,
min 200000 max 800000, i.e. 0.2-0.8 mm of travel). **Response**
(`ix_z_motion_rsp`): `position_nm` (min 199983 max 800113), `Exception`,
`Message`.

**Example**
```json
{"IxType":"ix_z_move","position_nm":800000,"IxID":17843}
```

#### `ix_focus_laser_set` / `ix_focus_laser_get` / `ix_focus_laser` (response)
Controls the (separate, low-power) focus laser. **Args** (set): `mode`
(`ix_focus_laser_mode_on`, `ix_focus_laser_mode_pwm`). **Response**: `mode`
(echoed), `Exception`, `Message`.

#### `ix_focus_laser_power_set`
Sets the focus laser's drive level. **Args**: `millivolts` (`100, 150,
500`). **Response** `ix_focus_no_rsp` — `Exception`, `Message` only.

#### `ix_focus_set_config`
Configures the focus-detection algorithm's camera/trigger parameters.

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `encoder_trigger` | int | `2132, 2133, 12288` | stage-encoder count at which to trigger a focus frame |
| `threshold` | int | `25` | spot-detection threshold |
| `gain` | int | `1` | |
| `image_save_enable` | bool | `False` | whether to persist focus frames for debugging |
| `exposure_camera_pulse_width_ns` | int | `250000` | |
| `exposure_laser_pulse_width_ns` | int | `27570, 250000` | |

**Response** `ix_focus_no_rsp` — `Exception`, `Message` only.

#### `ix_focus_set_model`
Configures the focus-tracking control-loop model.
**Args**: `gain` (`-100, -50`), `gain_boost` (`-100, -50`),
`boost_sample_count` (`4`), `best_delta_x` (int, min 3424576 max 6066070 —
units unclear, possibly a focus-error metric in camera sub-pixel units).
**Response** `ix_focus_no_rsp` — `Exception`, `Message` only.

#### `ix_focus_tracking_enable`
Enables/disables closed-loop autofocus tracking. **Args**: `enable` (bool,
`False` only in this run). **Response** `ix_focus_no_rsp` — `Exception`,
`Message` only.

#### `ix_focus_capture_frame` / `ix_focus_frame_info` (response payload)
Captures one focus-camera frame and returns its analysis. **Args**
(capture): `mode` (`ix_focus_camera_data_mode_image`). **Response**:
`y_position_nm`, `delta_x` (measured focus offset), `centroid_1`/`centroid_2`
(spot centroids — this focus system appears to track two spots, consistent
with a differential/astigmatic focus technique), `left_/right_avg_spot_inten`
and `left_/right_pct_spot_satur` (per-spot intensity and saturation),
`image_id`, `image_length_bytes` (`204000`), `z_position_nm`, `Exception`,
`Message`. Several fields show `max=4294967295` / `max=4294952959` — that's
`2^32-1`, i.e. an unsigned-32-bit "invalid/no signal" sentinel rather than a
real measurement; treat values at or near that maximum as "no spot found."

#### `ix_focus_etf_results_get` / `ix_focus_etf_results` (response payload)
Reads results from an ETF (focus-mapping) pass — ties to
[CIB's `ix_scan_etf`](CIB.md#ix_scan_etf). No data fields beyond the
envelope were captured in this run.

#### `ix_focus_histogram_get` / `ix_focus_histogram` (response payload)
Reads a focus-quality histogram/score. **Response**:
`histogram_one_percent` (`393150`), `histogram_ninetynine_percent`
(`400750`), `histogram_score` (`7600`), `Exception`, `Message`.

### Zoom optics

#### `ix_zoom_position_set` / `ix_zoom_position` (response payload)
Moves/reads a two-position zoom stage. **Args** (set): `position`
(`ix_zoom_pos_top`, `ix_zoom_pos_bottom`). **Response**: `position`
(echoed), `step_count` (int, min -4748 max 9651), `Exception`, `Message`.

**Example**
```json
{"IxType":"ix_zoom_position_set","position":"ix_zoom_pos_bottom","IxID":22689}
```

### Power monitoring

#### `ix_ilmn_pwr_voltage_monitor_get` / `ix_ilmn_pwr_current_monitor_get`
**Args**: `pwr_name` (`12v_pwr`, `24v_pwr`, `ftm_pwr` — "ftm" rail is unique
to FIB among boards seen so far, possibly "focus tracking module").
**Response**: min/max/avg `voltage_mV` (11936-24211) or `current_mA`
(123-1678 — notably higher than other boards, consistent with driving laser
diodes), `Exception`, `Message`.

### Diagnostics and logging (common firmware)

- **`ix_ix_echo_string`**, **`ix_io_get`**, **`ix_log_get_list`** — same as
  other boards.
- **`ix_log_replay`** — note the name: FIB uses `ix_log_replay` (no
  `_server` suffix), unlike FCH/RCA's `ix_log_replay_server`. Same purpose
  (stream a named log channel back), smaller channel list observed
  (`focus_servo` only in this run).

### Notification

#### `ix_io_no`
Spontaneous I/O pin-state push — see
[Commands vs. notifications](README.md#commands-vs-notifications).

**Args** (pushed payload): `ix_io_notify.name` — `shtr_open`, `shtr_close`
(laser shutter position feedback), `drvr_status` (laser driver status),
`fc_door0`, `fc_door1` (flow-cell door sensors — FIB apparently senses the
same doors [FCH](FCH.md#flow-cell-door) controls, from the optics side);
`state` (bool); `input_nOutput` (bool, `True` for all seen).

**Example**
```json
{"IxType":"ix_io_notify","IxID":-1,"name":"fc_door0","state":false,"input_nOutput":true}
```

## Open questions

- Whether "FIB" is Focus/Illumination or Fiber Illumination — both fit the
  command set; only the second half ("-IB") is guessed with any confidence.
- Units and interpretation of `ix_focus_set_model`'s `best_delta_x` and
  `ix_focus_frame_info`'s `delta_x`/`centroid_*` fields.
- Why `ix_lgm_power`'s observed max (`2100700` mW) is 1000x a plausible
  laser power — likely an uninitialized/error reading worth excluding from
  any derived "valid range."
- Relationship between `ix_focus_etf_results` here and `ix_scan_etf` on CIB
  — presumably one triggers a scan and the other reads its analyzed result,
  but the pairing isn't shown directly in this data.
