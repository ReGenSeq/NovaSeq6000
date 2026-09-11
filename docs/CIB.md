# CIB — Camera Interface Board

[← back to index](README.md)

**Confidence in name/role:** High. Response-type namespace is
`Illumina.Novaseq.IX.Cib_2` (same `_2` revision-suffix pattern as FCH's
`Fch_2`). The command set — an XY imaging stage, swath/ETF scan triggers,
raw camera frame capture, and (independently, outside the IX protocol —
see below) a DMA image-transfer service — is unambiguously the imaging
engine: this board positions the flow cell under the optics and pulls image
data off the camera(s) each sequencing cycle.

## Overview

CIB has two very different jobs visible in this repo's material:

1. **Over the IX protocol** (this doc): moving the XY stage, and issuing
   `ix_scan_swath`/`ix_scan_etf` to trigger a scan of one image swath —
   these calls carry all the imaging geometry (position, tilt, focus,
   cycle/surface/lane/swath identifiers) but, notably, **no response payload
   was captured for the scan-trigger commands themselves** (`response: None`
   in the library file) — the image data does not come back over this JSON
   channel.
2. **Outside the IX protocol**: `../cib_service_output.txt` (a separate log,
   from `CibService`, not from `logs/`) shows the actual image handoff — a
   DMA transfer of each swath's image data straight to RTA (Real-Time
   Analysis), timed independently of the IX request/response cycle (~1.2s
   per swath transfer in the sampled log, `78/78 images sent to RTA`).
   `../IPA_notes.txt` and `../dma_driver.txt` reference the same path
   (`rta3proxycib`: "implements receiving image swath data from CIB via DMA
   ... and sends data to RTA3"). **Treat the IX scan commands here as
   trigger/configuration only — the multi-gigabyte image payload itself
   moves over a separate DMA/shared-memory path, not JSON-over-TCP.**

Camera frame capture (`ix_camera_capture_frame`/`ix_camera_capture`) *does*
return real data over IX (`data_length_bytes`, `image_width/height/bpp`) —
that looks like a smaller, diagnostic single-frame capture path distinct
from full swath scanning, not the production imaging pipeline.

## Communication

See [protocol basics](README.md#protocol-basics) in the index. No CIB
endpoint IP was captured in this repo's material. The DMA image path
described above is a separate mechanism entirely (see `../cib_service_output.txt`)
and isn't part of the IX JSON protocol documented here.

## Commands

### XY stage motion

#### `ix_xy_move`
Moves one stage axis to an absolute position.

**Args**: `motor` (`ix_xy_motor_x`, `ix_xy_motor_y`), `position_nm` (int; X:
min -75000000 max 209500000; Y range not separately captured — see
`ix_xy_motion` response below for the combined range).
**Response** `ix_xy_motion_rsp` (see below).

**Example**
```json
{"IxType":"ix_xy_move","motor":"ix_xy_motor_x","position_nm":209500000,"IxID":17863}
{"IxType":"ix_xy_move","motor":"ix_xy_motor_y","position_nm":82500000,"IxID":17869}
```

#### `ix_xy_get_position` / `ix_xy_motion` (response payload)
Reads a stage axis' current position. **Args** (get): `motor`.
**Response**: `motor` (echoed), `position_nm` (int, min -74999998 max
209500078 across both axes), `Exception`, `Message`.

#### `ix_xy_motion_config_get` / `_set` / `ix_xy_motion_config` (response)
Reads/sets per-axis motion configuration (acceleration/velocity/etc. —
no config fields were captured beyond the envelope in this run).
**Args**: `motor` (only `ix_xy_motor_y` seen). **Response**
`ix_xy_motion_config_rsp` — `Exception`, `Message` only, in this run.

#### `ix_xy_motion_limits_set`
Sets soft motion limits for an axis. **Args**: `motor` (`ix_xy_motor_x`,
`ix_xy_motor_y` — no limit values were captured as scalar fields; likely
another case of list/nested-value args the parser drops, as noted on
[RCA's `ix_valve_config`](RCA.md#reagent-selector-valve)). **Response**
`ix_xy_no_rsp` — `Exception`, `Message` only.

### Scan triggering

#### `ix_scan_initialize`
Initializes the scan subsystem for a run. **Args**: `simulation` (bool,
`False` only). **Response** `ix_scan_empty_rsp` — `Exception`, `Message` only.

#### `ix_scan_swath`
Triggers a full-swath image scan at a given stage/optics configuration.
**No response payload captured** — see [Overview](#overview).

**Args**

| field | type | observed | notes |
|---|---|---|---|
| `id` | int | min 52, max 31519 | scan request id |
| `capture_start_nm` / `capture_end_nm` | int | 54750000–181250000 | Y-axis capture window along the flow cell |
| `image_start_nm` / `image_end_nm` | int | 54750000–181250000 | imaged sub-window within the capture window |
| `z_stage_init_method` | str | `ix_scan_z_stage_init_no_change`, `ix_scan_z_stage_init_z_start` | whether to re-home Z before this swath |
| `z_start_nm` | int | 308728–421969 | focus (Z) start position |
| `x_start_nm` | int | 142240207–208741949 | lane/tile X position |
| `tilt_enable_nDisable` | bool | both | whether to apply tip/tilt correction for this swath |
| `tilt_v_nm` / `tilt_cone_nm` / `tilt_flat_nm` | int | ~5.4M–5.7M | tilt-motor targets for this swath — same 3-axis scheme as [FCH's tip/tilt motors](FCH.md#flow-cell-tip-tilt-alignment); CIB requests the tilt values that go with each swath, FCH's motors execute them |
| `cycle` | int | 0–318 | sequencing cycle number |
| `surface` | int | `1`, `2` | flow-cell surface (top/bottom) |
| `fc_number` | int | `1`, `2` | which flow cell |
| `lane_id` | int | `0`–`4` | flow-cell lane |
| `swath_id` | int | `0`–`6` | swath index within the lane |
| `focus_servo_enable_nDisable` | bool | both | live autofocus on/off for this swath |
| `focus_lead_nm` | int | `0`, `1250000` | focus servo lead/lookahead distance |
| `target_for_de` | bool | both | "de" likely = dynamic exposure; whether this swath is a target for that feature |

**Example**
```json
{"IxType":"ix_scan_swath","id":53,"capture_start_nm":67417000,"capture_end_nm":69417000,"image_start_nm":67417000,"image_end_nm":69417000,"z_stage_init_method":"ix_scan_z_stage_init_z_start","z_start_nm":410543,"x_start_nm":142617000,"tilt_enable_nDisable":true,"tilt_v_nm":5438750,"tilt_cone_nm":5498750,"tilt_flat_nm":5707500,"cycle":0,"surface":2,"fc_number":1,"lane_id":0,"swath_id":0,"focus_servo_enable_nDisable":false,"focus_lead_nm":0,"target_for_de":false,"IxID":22716}
```

#### `ix_scan_etf`
A second scan-trigger command with the same general shape as
`ix_scan_swath` but a narrower, fixed-looking window (`z_etf_delta_nm: 52`
suggests "ETF" = a focus-calibration or extended-tile-focus scan, distinct
from a normal imaging swath — likely a focus-mapping pass rather than
data-collection imaging). Same caveat: no response payload captured.

#### `ix_scan_queued_no` (notification, not a command)
A spontaneous push with a large `id` set — reads as the board's queue of
scan-request ids it has accepted/queued, not a response to any single
request. See [Commands vs. notifications](README.md#commands-vs-notifications).

#### `ix_scan_cancel`
Cancels a pending/in-progress scan. **Response** `ix_scan_empty_rsp` —
`Exception`, `Message` only.

### Camera (diagnostic single-frame capture)

#### `ix_camera_capture_frame` / `ix_camera_capture` (response payload)
Captures a single raw frame from the camera — separate from the swath scan
path (see [Overview](#overview)).

**Args**: `capture_id` (int, `3`, `4`), `data_mode`
(`ix_camera_data_mode_image`), `exposure_usec` (`3000`), `bitdepth`
(`ix_camera_bpp_16`).
**Response** `ix_camera_capture_rsp`: `capture_id` (echoed),
`data_length_bytes` (`524800`), `image_width` (`3200`), `image_height`
(`82` — much shorter than a full swath, consistent with a small diagnostic
frame), `image_bpp` (`ix_camera_bpp_16`), `Exception`, `Message`.

### Power monitoring

#### `ix_ilmn_pwr_voltage_monitor_get` / `ix_ilmn_pwr_current_monitor_get`
**Args**: `pwr_name` (`xy_pwr`, `camera_pwr`). **Response**: min/max/avg
`voltage_mV` (22296–23272) or `current_mA` (3–10), `Exception`, `Message`.

#### `ix_ilmn_pwr` (notification, not a command)
Spontaneous power-rail push — see
[Commands vs. notifications](README.md#commands-vs-notifications).
`pwr_name` seen: `xy_pwr`.

### Diagnostics (common firmware)

- **`ix_ix_echo_string`** — loopback test, same as every other board.

## Open questions

- What the image DMA path (`CibService`, `dma_driver.txt`,
  `rta3proxycib`) looks like from software's side beyond what the two log
  excerpts show — this doc only covers the JSON/IX control channel.
- Exact meaning of `ix_scan_etf` ("ETF") and how it's sequenced relative to
  `ix_scan_swath` in a real cycle.
- Field values for `ix_xy_motion_limits_set` — likely dropped by the parser
  the same way `ix_valve_config`'s `valid_positions` was (see RCA doc).
- Whether `target_for_de` really refers to "dynamic exposure" — a guess from
  the abbreviation alone.
