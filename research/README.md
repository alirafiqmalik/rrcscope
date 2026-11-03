# research/

[← Project README](../README.md)

Write-ups of captures made with this toolchain. Each entry is **frozen**: it records
what one session showed. The maintained code is in [`scripts/`](../scripts/) and
[`src/nasrrc/`](../src/nasrrc/).

| Study | Date | Summary |
|---|---|---|
| [LTE attach capture](lte-attach-capture/README.md) | 2026-08-13 | A full detach and combined EPS/IMSI re-attach while roaming. Every message is tagged with its on-air protection, and the IMEISV goes out in a plain-NAS Identity Response before security |

Device-level measurements from building the live path (latency, the DM node
mirror, platform limits) are in [docs/findings.md](../docs/findings.md).

## Rules for adding a study

- Mask permanent identifiers. Show only the 8-digit TAC of an IMEI/IMEISV
  (`35497743••••••••`), and leave out IMSI, MSISDN, TMSI/GUTI, and SMS bodies.
- Put no raw `.sdm`, `.pcap`, or `.qmdl` files in git. A summary
  (`frame / time / protocol / Info`) has no identifiers, so it is acceptable.
- Write down the device, the modem firmware if known, the network, and the date,
  and put the diagram inline as Mermaid so GitHub renders it.
