# NovaSeq 6000 embedded device documentation

This folder documents the "IX" JSON command protocol the NovaSeq 6000 instrument
controller uses to talk to the embedded controller boards inside the instrument
(fluidics, imaging, thermal, reagent handling, etc). It exists to make the
`*_library.md` files in the repo root — which are raw, auto-parsed command
listings — readable and usable as a reference for `NovaPySeq6000`.

## Source material and how it was produced

- `logs/` contains the raw instrument log files from one real sequencing run
  (run `SNA00404_0666`, flow cell `BHF2HYDSX7`, 2023-08-04, side B only).
- `parse_logs.py` scans those logs for lines matching `IX (Tx|Rx) (id): DEVICE {json}`,
  groups them by embedded device (`DEVICE`) and command (`IxType`), and writes one
  `<DEVICE>_library.md` file per device summarizing every argument and response
  field it saw, with the *set* of observed values (or a min/max range once more
  than 10 distinct values appear).
- `command_library.md` is the same data with all devices concatenated into one file.

**This means every value in the library files, and in the docs here, is what one
run happened to exercise — not a documented spec.** A `min=20000, max=60000` on a
temperature argument is the range this run used, not necessarily the hardware's
actual limits, and an enumerated set like `{'wsv_side_b'}` only has one member
because side A was never active in this run. Treat ranges/sets as a *lower bound*
on what's valid, and flag anything you rely on for correctness against the real
instrument.

`parse_logs.py` also only records **scalar** argument values — list- and
object-valued arguments (a valve's `valid_positions` list, an RFID tag's full
`data`/`locked` arrays, motion limit values) are silently dropped from the
library `.md` files. Several are recovered and documented here (see RCA's
`ix_valve_config`, CHM's `ix_rfid_tag_set`) by reading `logs/` directly — if
you need a field and it's missing from a library file, check whether it's a
list/object value the parser dropped before assuming the board doesn't send it.

## Protocol basics

- **Transport**: each embedded board is a JSON-over-TCP server on port `5555`
  (see `../command_example.py`). Endpoints seen so far are link-local
  (`169.254.x.x`) addresses, consistent with a dedicated point-to-point Ethernet
  link per board rather than a shared LAN segment.
- **Request** (host → board): a single JSON object —
  `IxType` (command name), zero or more command-specific arguments, `ResponseType`
  (the .NET-qualified type name the host expects back, e.g.
  `Illumina.Novaseq.IX.Flu.ix_valve_set_position_rsp, Illumina.Novaseq.IX.Flu, ...`),
  and `IxID` (a caller-assigned integer request id).
- **`IxID` is required.** `command_example.py` shows that a request without it is
  rejected first with `{"IxType":"ix_internal_error_notification", ...,
  "Exception":"ix_exception_id_command_parsing", "Message":"Couldn't find
  command's IxID parameter"}` — the board then still executes the command using
  `IxID: 0`. Always send a unique `IxID` and match it against the response.
- **Response** (board → host, same `IxID`): `IxType` (`<command>_rsp`), `IxID`
  (echoed), zero or more result fields, `Exception` (a status code — normally
  `ix_exception_id_success`; other values seen include
  `ix_exception_id_errno_error`, `ix_exception_id_errno_state_trans`,
  `ix_exception_id_generic_failure`, and the parsing error above), and `Message`
  (empty on success, a short human-readable string otherwise, e.g. `"Invalid
  state transition"`, `"An error has occurred while executing the command"`).

### Commands vs. notifications

The library files mix two different message shapes under one format, and it's
worth telling them apart:

- **Commands** — a host→board `Tx` that gets a matching `Rx` of type
  `<command>_rsp` with real content. These show up in the library files as
  `# ix_command_name` / `## response: ix_command_name_rsp` with response fields.
- **Notifications** — messages the board pushes to the host on its own, not in
  response to a request. In the raw logs these arrive as `IX Rx (0): DEVICE
  {"IxType":"ix_io_notify", "IxID":-1, ...}` — `IxID: -1` and no prior `Tx`, i.e.
  spontaneous. `parse_logs.py` files these under the *bare* `IxType` it sees
  (`ix_io_no`, `ix_ilmn_pwr`) with `response: None`, because it never finds a
  paired `_rsp`. Confirmed for `ix_io_no` (I/O pin transitions — door sensors,
  limit switches, presence switches) and `ix_ilmn_pwr` (periodic power-rail
  telemetry) on every device that has them. Don't try to "call" these —
  they're read-only status streams.

### Common firmware commands vs. board-specific commands

Response-type namespaces split commands into two families:

- **Board-specific** commands use a namespace tied to that board, e.g.
  `Illumina.Novaseq.IX.Flu` (FLU), `Illumina.Novaseq.IX.Fch_2` (FCH),
  `Illumina.Novaseq.IX.Bim` (BIM), `Illumina.Novaseq.IX.Rca` (RCA),
  `Illumina.Novaseq.IX.Syst` (SYST), `Illumina.Novaseq.IX.Cib_2` (CIB),
  `Illumina.Novaseq.IX.Fib_2` (FIB). These are the commands unique to that
  board's job (valves, thermal zones, motors, lasers, ...). The `_2` suffix
  on FCH/CIB/FIB (but not FLU/BIM/RCA/SYST) is a consistent pattern across
  the data but its meaning (hardware/protocol revision?) isn't confirmed.
  CHM showed no board-specific namespace at all in this run — see
  [CHM.md](CHM.md).
- **Common firmware** commands share one namespace regardless of which board
  answers them — `Illumina.Voyager.IX.IxDevices.CommonCommands.Firmware.*,
  Illumina.NovaSeq.IX, Version=1.7.5.27` — e.g. `ix_ix_echo_string`, `ix_io_get`,
  `ix_rfid_tag_get`/`ix_rfid_tag_set`, `ix_log_get_list`,
  `ix_log_replay_server` (`ix_log_replay` on FIB — see its doc),
  `ix_fpga_counters_reset`. These look like a shared embedded-firmware base
  every board runs, independent of the board's role. `Version=1.7.5.27` here
  vs `0.0.0.0` on board-specific commands is also a useful tell for which
  family a command belongs to.

## Devices

| Board | Best-guess full name | Confidence | Physical role (inferred) | Doc |
|---|---|---|---|---|
| FLU | Fluidics board | High — namespace confirmed | Reagent pumps, wash-selector valves, pressure/flow sensors, pump/degasser power, for the active flow-cell side | [FLU.md](FLU.md) |
| FCH | Flow Cell Holder | Medium-high | Flow-cell thermal control, tip/tilt alignment motors, vacuum hold-down, hold engage/disengage, FC door, status lightbar, FC RFID tag — serves both left and right flow-cell bays from one board | [FCH.md](FCH.md) |
| BIMLeft / BIMRight | Bulk (reagent bottle) Input Module — Left/Right | Medium — namespace confirmed, role inferred | Raises/lowers sippers into bulk reagent source bottles (`ix_fluidics_automation_do op up/down`); bottle-presence and handle sensors (`sour1_present`, `sour2_present`, `sour2_handle_down`) | [BIM.md](BIM.md) |
| RCALeft / RCARight | Reagent Chiller Assembly — Left/Right | Medium-high — namespace confirmed | Reagent cartridge/tray chiller: cartridge presence (`cart_a1`/`cart_a2`), sipper up/down, sequencing/wash tray doors, chiller fan, thermal zone (`chiller0`), 24-position reagent selector valve, pierce/mix/wash automation | [RCA.md](RCA.md) |
| FIB | Focus & Illumination Board | Medium | Excitation laser power/mode/shutter control (red/green, via "LGM"), a full autofocus loop (focus laser, Z motor, focus camera, focus scoring), and a top/bottom zoom stage | [FIB.md](FIB.md) |
| CIB | Camera Interface Board | High | Imaging XY stage motion, swath/ETF scan triggering, single-frame diagnostic camera capture; production image data moves over a separate DMA path to RTA, not over IX (confirmed via `../cib_service_output.txt`) | [CIB.md](CIB.md) |
| CHM | Chemistry (buffer) Module | Medium — no board-specific namespace observed | Buffer-bottle compartment: RFID tag read/write on two buffer bottles, two drawer doors | [CHM.md](CHM.md) |
| SYST | System controller | High — namespace confirmed | Chassis-wide monitoring: fans across subsystems, compressor current, pump current, 12V/24V power rails, a liquid-level sensor | [SYST.md](SYST.md) |

"Confidence" reflects how sure the *name and role* are, not the *data* — the
argument/response tables for every device are transcribed directly from the
library files (plus a few fields recovered from `logs/` where the parser
dropped list/object values) and are only as complete as the one captured run.

## Doc template

Each device doc follows the same shape:

1. **Overview** — physical role, confidence in that role, anything the naming
   or command set implies about the hardware.
2. **Communication** — endpoint info if known, anything device-specific about
   the protocol.
3. **Commands**, grouped by function, each with:
   - a one- or two-sentence description of what it does (flagged if inferred),
   - an args table (field, type, observed values/range, notes),
   - a response table for true command/response pairs,
   - a real Tx/Rx example pulled from `logs/`, where one exists.
4. **Notifications** — any spontaneous board→host messages, documented
   separately from commands.
5. **Open questions** — anything left uncertain for a human to confirm.

## Cross-device patterns worth knowing

A few things only became clear by comparing devices, and are easy to miss
reading one doc in isolation:

- **Left/Right doesn't always mean flow-cell side.** For BIM and FCH, `left`/
  `right` (or a board split, for BIM) lines up with the two flow-cell sides.
  For RCA, the two boards' *data* points to two different cartridge bays
  (buffer vs. reagent) rather than a left/right flow-cell split — see
  [RCA.md](RCA.md#left-vs-right-two-different-cartridge-bays-not-two-flow-cell-sides).
- **Tip/tilt and scan geometry span two boards.** FCH owns the physical
  tip/tilt motors; CIB's `ix_scan_swath` sends the tilt targets that go with
  each swath. Reading FCH's and CIB's docs together gives the fuller picture
  of flow-cell alignment during imaging.
- **Focus spans two boards too.** FIB runs the autofocus loop (focus laser,
  Z motor, focus camera); CIB's `ix_scan_etf` looks like the scan-triggered
  counterpart to FIB's `ix_focus_etf_results`.
- **RFID is one common-firmware command, several physical tag readers.**
  FCH (flow cell), RCA (reagent cartridges), and CHM (buffer bottles) all
  call the same `ix_rfid_tag_get`/`_set`, just with different `rfid` values.
- **Status codes worth checking for, not just `success`:**
  `ix_exception_id_errno_error`, `ix_exception_id_errno_state_trans` (FCH,
  invalid hold-state transition), `ix_exception_id_generic_failure` paired
  with `ix_rfid_error_status_failed_inventory` (RCA, a tag that failed to
  read) — these are the only non-success cases captured in this run, and are
  worth testing for explicitly rather than assuming every response is a
  clean success.

## Status

All 10 embedded devices (8 docs, two of which each cover a left/right pair)
are documented as of this pass. Each doc's "Open questions" section lists
what's inferred rather than confirmed — worth a pass against real hardware,
Illumina documentation if available, or a wider set of run logs (this
material only comes from one side-B-only run) before treating anything here
as authoritative for anything safety- or correctness-critical.
