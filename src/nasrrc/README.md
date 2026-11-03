# `nasrrc` — the analyzer package

[← Project README](../../README.md)

This package turns GSMTAP NAS/RRC into control-plane and mobility events. It uses only the Python
standard library, and it runs `tshark` as a subprocess. It does not need a phone:
it reads a pcap, a FIFO, a live interface, or a tshark summary file.

| Module | Role |
|---|---|
| [`fields.py`](fields.py) | **Field analyzer.** Parses tshark PDML as a stream and keeps measId → trigger state. Gives measurement reports with per-cell levels, and handovers that are really handovers |
| [`events.py`](events.py) | **Summary classifier.** Matches the tshark Info column against an ordered rule table. Fast, but it cannot tell a handover from other reconfigurations |
| [`cli.py`](cli.py) | `python -m nasrrc` / `nasrrc-analyze` |
| [`__init__.py`](__init__.py) | Public API |

## CLI

```text
python -m nasrrc [input] [-i IFACE] [--port 4729] [--host 127.0.0.1] [--fields] [--kinds] [--json]
```

| Input | Mode used |
|---|---|
| `cap.summary.txt` | Summary classifier |
| `cap.pcap` | Summary classifier (Info column through tshark) |
| `cap.pcap --fields`, or a FIFO with `--fields` | Field analyzer |
| `-i lo` | Field analyzer on a live capture (`--fields` is implied) |

`--kinds` prints counts, and `--json` prints one object per line. A broken pipe
(for example `| head`) and Ctrl-C both end the run cleanly. If tshark exits with
an error, the CLI reports it. Without this, a capture that never started would
look the same as a quiet radio.

```console
$ python -m nasrrc --fields --kinds fixtures/synthetic-lte/lte-mobility.pcap
    3  rrc_reconfiguration
    2  rrc_reconfiguration_complete
    2  measurement_report
    1  attach_request
    1  identity_request
    1  identity_response
    1  tau_request
    1  handover
```

## Library

```python
from pathlib import Path
from nasrrc import analyze_pcap

for ev in analyze_pcap(Path("fixtures/synthetic-lte/lte-mobility.pcap")):
    if ev.kind == "measurement_report":
        best = max(ev.cells, key=lambda c: c.rsrp_dbm or -999)
        print(ev.frame, ev.trigger, best.role, best.pci, best.rsrp_dbm)
    elif ev.kind == "handover":
        print(ev.frame, "handover ->", ev.target_pci)
```

```text
9 a3 neighbour 250 -70.0
10 b1 serving None -83.0
11 handover -> 250
```

| Entry point | Use |
|---|---|
| `analyze_pcap(path)` | A file or a FIFO that carries a pcap stream |
| `analyze_interface(iface, port=4729, host=None)` | Live capture through tshark `-l` |
| `analyze_pdml(chunks, analyzer=None)` | Any iterable of PDML text. Useful for tests and other transports |
| `Analyzer().feed(packet)` | One PDML `<packet>` element → `FieldEvent` or `None` |
| `classify_summary_file(path)` / `classify_summary_line(line)` | Summary classifier |

Each generator stops its tshark process when the consumer stops iterating. It
closes the read end first, because a tshark that is blocked on a full pipe never
gets to its own signal handler.

## Event schema

`FieldEvent.as_dict()` — this is the `--json` output for frame 9 of the synthetic fixture:

```json
{
  "frame": 9, "t_rel": 0.8, "protocol": "lte_rrc", "rat": "lte",
  "kind": "measurement_report", "message": "measurementReport",
  "meas_id": 1, "trigger": "a3",
  "target_pci": null, "target_freq": null, "meas_config": [],
  "cells": [
    {"role": "serving",                "rsrp": 62, "rsrq": 24, "rsrp_dbm": -79.0, "rsrq_db": -8.0},
    {"role": "neighbour", "pci": 250,  "rsrp": 71, "rsrq": 27, "rsrp_dbm": -70.0, "rsrq_db": -6.5},
    {"role": "neighbour", "pci": 301,  "rsrp": 55, "rsrq": 18, "rsrp_dbm": -86.0, "rsrq_db": -11.0}
  ]
}
```

| Field | Set on |
|---|---|
| `meas_id`, `trigger`, `cells` | `measurement_report`. `trigger` is `a1`…`a6`, `b1`, `b2`, `periodical`, or `null` if the configuration came before the capture started |
| `target_pci`, `target_freq` | `handover`, `inter_rat_handover` |
| `meas_config` | Any reconfiguration that adds measIds: a list of `[measId, trigger]` |
| `cells[].serv_cell_id`, `sinr`, `sinr_db` | NR only. An EN-DC report has one serving entry per measurement object |

## Event kinds

| Kind | Source | Rule |
|---|---|---|
| `handover` | LTE/NR RRC | A reconfiguration that carries `mobilityControlInfo` (LTE) or `reconfigurationWithSync` (NR) |
| `inter_rat_handover` | LTE/NR RRC | `mobilityFromEUTRACommand` / `mobilityFromNRCommand` |
| `measurement_report` | LTE/NR RRC | `measurementReport` |
| `rrc_reconfiguration` / `_complete` | LTE/NR RRC | Any other reconfiguration |
| `rrc_connection_request` / `_setup` / `_setup_complete` | LTE/NR RRC | Including NR `rrcSetup` |
| `security_mode_command` / `_complete` | RRC | AS security mode |
| `attach_*`, `identity_*`, `esm_information_*`, `tau_*`, `detach_request`, `registration_*` | NAS-EPS / NAS-5GS | Rule table in `events.py`. The first match wins, and the more specific strings come first |
| any other dedicated RRC message | RRC | The message name in snake_case, e.g. `ue_capability_enquiry` |

BCCH and PCCH (system information and paging) are dropped, because they carry no
mobility events.

## Signal-level conversions

The raw values are the ASN.1 range indices that tshark shows. Each conversion gives
the lower bound of the reported range, as 36.133 / 38.133 define it and as Wireshark
labels it.

| Quantity | LTE (36.133) | NR (38.133) |
|---|---|---|
| RSRP | `-141 + v` dBm | `-157 + v` dBm |
| RSRQ | `-20 + v/2` dB | `-43.5 + v/2` dB |
| SINR | — | `-23.5 + v/2` dB |

Check against Wireshark's labels on device data: NR rsrp 43 → −114 dBm,
rsrq 63 → −12.0 dB, sinr 64 → 8.5 dB.

## Design notes

- **PDML, not `-T fields`.** `reportConfigId` occurs in two different lists. Flat
  field output loses which list each value came from, so the measId → event mapping
  cannot be recovered. See [docs/architecture.md](../../docs/architecture.md#the-analyzer).
- **Suffix matching.** A field is found by the end of its name
  (`…mobilityControlInfo_element`), so one code path reads both `lte-rrc.*` and
  `nr-rrc.*`.
- **Innermost protocol, hidden ones skipped.** NAS inside RRC `dedicatedInfoNAS` is
  classified as NAS. NR RRC nested in an LTE EN-DC reconfiguration is `hide="yes"`
  and is skipped, so the message stays one LTE event.
- **Remove before add.** `measIdToRemoveList` is applied before
  `measIdToAddModList`, as 36.331 specifies.
