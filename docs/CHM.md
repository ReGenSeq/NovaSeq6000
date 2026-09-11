# CHM — Chemistry (buffer) Module

[← back to index](README.md)

**Confidence in name/role:** Medium. No board-specific namespace was seen
for CHM in this run's traffic (every CHM command captured is a
common-firmware command — RFID, IO, echo) so there's nothing to confirm the
name against directly, unlike FLU/FCH/BIM/RCA/SYST. The role is inferred
purely from the command set: two RFID-tagged buffer bottle slots behind two
drawer doors. "Chemistry Module" is a guess at what "CHM" expands to;
"buffer compartment" describes what it does with more confidence than the
name itself.

## Overview

CHM is the smallest device documented — every command it exercised in this
run is one shared with other boards (see
[common firmware commands](README.md#common-firmware-commands-vs-board-specific-commands)),
there's no CHM-specific namespace/command set the way FLU has `ix_pump_*` or
FCH has `ix_thermal_*`. What it does is inferred entirely from *which*
common commands it uses and their arguments: reading/writing RFID tags named
`buffer_1`/`buffer_2`, and two drawer-door sensors (`draw_door1`,
`draw_door2`) plus a bottle-present sensor (`buffer_present1`). That's a
small compartment holding up to two buffer bottles, each identified by an
RFID tag, behind doors.

**Open question**: since no CHM-specific namespace appears in this run, it's
possible CHM has board-specific commands (temperature control for a buffer
chiller zone, for instance) that simply weren't exercised — this run may
have only ever read the tags and watched the doors. Don't take "no
board-specific commands observed" as "none exist."

## Communication

See [protocol basics](README.md#protocol-basics) in the index. No CHM
endpoint IP was captured in this repo's material.

## Commands

All of CHM's captured commands are **common firmware** commands (namespace
`Illumina.Voyager.IX.IxDevices.CommonCommands.Firmware.*,
Illumina.NovaSeq.IX, Version=1.7.5.27`), shared with other boards — see
[FCH's RFID section](FCH.md#flow-cell-rfid) for the general shape.

#### `ix_rfid_tag_get` / `ix_rfid_tag_set`
Reads/writes a buffer bottle's RFID tag.

**Args**: `rfid` (`buffer_1`, `buffer_2`).

**`ix_rfid_tag_set` also carries a full `tag` object** that
`CHM_library.md` doesn't show at all — `parse_logs.py` only records
scalar argument values (see the same caveat on
[RCA's `ix_valve_config`](RCA.md#reagent-selector-valve) and
[CIB's `ix_xy_motion_limits_set`](CIB.md#xy-stage-motion)), so a
nested-object argument like `tag` is silently dropped from the library file.
Recovered directly from `logs/`, the real shape is:

```json
"tag": {
  "IxType": "ix_rfid_tag",
  "rssi": 0,
  "uid": 16143013194396750096,
  "data": [1543569676, 1241580801, 73729, /* ... 28 uint32 words total */],
  "locked": [false, false, /* ... 28 booleans, one per data word */]
}
```

This is the tag's actual memory contents: `uid` (the tag's unique id),
`data` (28 32-bit words — likely lot/expiry/cartridge-identity data written
by Illumina at manufacture and read back here), and `locked` (a per-word
write-protect flag). If you're implementing tag writes, replicate this
whole structure, not just `rfid`.

**Response** `ix_rfid_tag_get_rsp` / `ix_rfid_tag_set_rsp`: `error`
(`ix_rfid_error_status_noerr` in every CHM instance captured — CHM's tag
reads didn't show the `failed_inventory` error RCA's did), `Exception`,
`Message`.

**Example**
```json
// get
{"IxType":"ix_rfid_tag_get","rfid":"buffer_1","IxID":1125}
// set (tag payload abbreviated - see full shape above)
{"IxType":"ix_rfid_tag_set","rfid":"buffer_2","tag":{"IxType":"ix_rfid_tag","rssi":0,"uid":16143013194396750096,"data":[...28 words...],"locked":[...28 bools...]},"IxID":17905}
```

### Diagnostics (common firmware)

- **`ix_io_get`**, **`ix_ix_echo_string`** — same as every other board.

### Notification

#### `ix_io_no`
Spontaneous I/O pin-state push — see
[Commands vs. notifications](README.md#commands-vs-notifications).

**Args** (pushed payload): `ix_io_notify.name` — `draw_door1`, `draw_door2`
(the two buffer-drawer doors), `buffer_present1` (bottle-presence sensor —
only one `_present` sensor observed despite two buffer slots; either
`buffer_present2` simply wasn't toggled in this run, or one slot lacks a
presence sensor); `state` (bool); `input_nOutput` (bool, `True` for all
seen).

**Example**
```json
{"IxType":"ix_io_notify","IxID":-1,"name":"draw_door1","state":false,"input_nOutput":true}
```

## Open questions

- What "CHM" expands to — no board-specific namespace to check it against.
- Whether CHM has board-specific commands (e.g. a buffer chiller zone) not
  exercised in this run.
- Whether there's a `buffer_present2` sensor that just didn't fire in this
  run, or whether slot 2 genuinely has no presence sensor.
- What the 28 `data` words in an RFID tag encode (lot number, expiry,
  cartridge type, checksum, etc. are all plausible; nothing here decodes
  them).
