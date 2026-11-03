# fixtures/

[← Project README](../README.md)

Real captures contain subscriber identities, so they are not in git (see
`.gitignore`). The analyzer is developed and tested against these files.

## `synthetic-lte/` — hand-encoded GSMTAP pcap

[`make_pcap.py`](synthetic-lte/make_pcap.py) encodes LTE RRC and NAS messages by
hand in UPER, wraps each message in GSMTAP/UDP/IPv4, and writes
[`lte-mobility.pcap`](synthetic-lte/lte-mobility.pcap). tshark dissects all twelve
frames without an expert warning. The pcap contains no subscriber data. It is the
only pcap that `.gitignore` lets into the repo.

| # | Channel | Message | What it exercises |
|--:|---|---|---|
| 1 | NAS-EPS | Attach request + PDN connectivity request | NAS classification |
| 2 | NAS-EPS | Identity request | |
| 3 | NAS-EPS | Identity response | |
| 4 | NAS-EPS | Tracking area update request | |
| 5 | DL-DCCH | RRCConnectionReconfiguration — reportConfig 1 = **eventA3**, measId 1 | measId → trigger learning |
| 6 | DL-DCCH | RRCConnectionReconfiguration — reportConfig 2 = **eventB1** (inter-RAT), measId 2 | |
| 7 | DL-DCCH | RRCConnectionReconfiguration — no measConfig, no mobilityControlInfo | **not** a handover |
| 8 | UL-DCCH | RRCConnectionReconfigurationComplete | complete ≠ reconfiguration |
| 9 | UL-DCCH | MeasurementReport measId 1 — serving + neighbours PCI 250, 301 | per-cell RSRP/RSRQ |
| 10 | UL-DCCH | MeasurementReport measId 2 — serving only | |
| 11 | DL-DCCH | RRCConnectionReconfiguration + **mobilityControlInfo** → PCI 250 | handover detection |
| 12 | UL-DCCH | RRCConnectionReconfigurationComplete | |

```bash
.venv/bin/python fixtures/synthetic-lte/make_pcap.py   # regenerate
./scripts/dump_pcap.sh fixtures/synthetic-lte/          # look at it
```

The bit layouts follow 36.331 as Wireshark 4.7 dissects it. There is one deviation
from the published grammar: Wireshark's `MeasResults` has **two** root optional bits
before `measId`, and only the second one gates `measResultNeighCells`.

## `example-lte/` — a real attach, summary only

[`LTE_full_summary.txt`](example-lte/LTE_full_summary.txt) is the tshark
`frame / time / protocol / Info` summary of a real LTE attach (202 frames). It
contains only message names: MIB/SIBs, RRC setup, attach, identity, ESM information,
security mode, UE capability, reconfigurations, measurement reports, and paging. It
contains no identifiers, so it can be in git. It is the input for the summary
classifier tests:

```bash
python -m nasrrc --kinds fixtures/example-lte/LTE_full_summary.txt
```
