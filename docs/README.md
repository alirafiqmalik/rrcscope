# Docs

[← Project README](../README.md)

Topics that apply to more than one directory. Each code directory has its own README for its details.

| Document | Read it for |
|---|---|
| [architecture.md](architecture.md) | How a message travels from the modem to your terminal: both live paths, SCAT parser selection, radio filtering, the analyzer's state |
| [findings.md](findings.md) | What the Pixel 7a really did: latency, throughput, platform limits, results on live data |
| [live-analysis-handoff.md](live-analysis-handoff.md) | The original design note (2026-08): why SCAT and not MobileInsight, and the first build plan |
| [assets/](assets/) | Images used by the READMEs. `analyzer-demo.svg` is real output of `live_analyze.sh` on the synthetic fixture; `social-preview.png` is the 1280×640 repository card |

## Directory READMEs

| Directory | README |
|---|---|
| `scripts/` | [Host and device scripts](../scripts/README.md) |
| `src/nasrrc/` | [Analyzer package and CLI](../src/nasrrc/README.md) |
| `tests/` | [Test suite](../tests/README.md) |
| `fixtures/` | [Test captures](../fixtures/README.md) |
| `research/` | [Research write-ups](../research/README.md) |

## Development records

Decisions and tickets are kept as Markdown in [`.scratch/`](../.scratch/) because
the project's agent workflow reads them from there:

- [ADR-0001 — keep it simple](../.scratch/adr/0001-keep-it-simple.md)
- [Spec — live NAS/RRC analyzer](../.scratch/live-analyzer/spec.md)
- [Engineering log (map)](../.scratch/live-analyzer/map.md)
- Tickets: [01 ring path](../.scratch/live-analyzer/issues/01-prove-tail-the-ring.md) ·
  [02 analyzers](../.scratch/live-analyzer/issues/02-handover-measurement-analyzers.md) ·
  [03 DM node](../.scratch/live-analyzer/issues/03-scat-owns-diag.md) ·
  [04 node as live path](../.scratch/live-analyzer/issues/04-node-tap-as-live-path.md)
