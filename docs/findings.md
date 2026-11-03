# Device findings

[← Docs index](README.md) · [← Project README](../README.md)

What a rooted Pixel 7a actually did, measured during development (2026-09-02
unless noted). These results drove the design in [architecture.md](architecture.md).
The full engineering log, with each failed attempt, is
[`.scratch/live-analyzer/map.md`](../.scratch/live-analyzer/map.md).

## Test device

| | |
|---|---|
| Phone | Google Pixel 7a (`lynx`), rooted, Android `user` build |
| SoC / modem | Tensor GS201 · Samsung Exynos Modem 5300 (`g5300q`, Shannon) |
| Diagnostic format | SDM (SCAT `-t sec`), not Qualcomm DIAG |
| DM node | `/dev/umts_dm0`, held by `dmd` (init service `DM-daemon`) |
| Host decoder | SCAT / `signalcat` 2.0 |
| Host dissector | Wireshark / tshark 4.7.2 |

## Measured numbers

| Measurement | Value |
|---|---|
| Ring path: SCAT attach → first GSMTAP packet | **2.75 s** (5.71 s from script launch) |
| Ring path: steady-state burst spacing | 5.5–6.6 s (follows dmd's flush/rotation cadence) |
| Ring rotation interval | ~9 s observed; ~30 s with a 52 MB ring |
| DM node tap: median inter-arrival | **44 ms** |
| DM node tap: longest gap (idle, stationary UE) | 8.1 s |
| DM node tap: throughput, 60 s with a session ring | 74,547,085 bytes |
| DM node tap: `cat` iterations under load | ~5 per second |
| dmd ring growth while the tap read 9.1 MB in 25 s | 14,685,987 → 20,981,219 bytes (nothing taken from dmd) |
| Ring path, 6 rotations followed | 1 remote tail alive throughout, 0 leaked |

## On live data

- **425 GSMTAP packets → 81 events**: 63 measurement reports, 9 reconfigurations,
  9 completes, and 0 handovers. Zero handovers is correct for a stationary UE.
- **measId → trigger works on a live network.** The network re-sends the NR
  measurement config about once a minute
  (`configures measId26=a1,measId27=a2,measId28=a5,measId29=a5`), and the next
  report arrives as `measId=26 trigger=a1`. An LTE measId that was configured
  before the capture started correctly has no trigger.
- **First real handover:** an EN-DC `RRCConnectionReconfiguration` with
  `mobilityControlInfo` to target PCI 90 and a nested NR `reconfigurationWithSync`.
  The analyzer reports it once, as an LTE handover.
- **EN-DC reports have two serving MOs** for the same PSCell (servCellId 7 and 8).
  For this reason the analyzer reports every serving MO and tags each one with its servCellId.
- **The conversions agree with Wireshark's own labels:** NR rsrp 43 → −114 dBm,
  rsrq 63 → −12.0 dB, sinr 64 → 8.5 dB. LTE values agree with SCAT's PHY log for the
  serving PCI 90 (−105 to −113 dBm in the same window).

## Constraints of the platform

- **About one logging session per boot.** After `start modem_logging_stop`, no
  `start modem_logging_start` restarts it; a reboot is required. `sys.boot_completed`
  still reads `true` from before the reboot, so wait for the adb disconnect instead.
- **`vendor.sys.modem.logging.status` is not a readiness signal.** It read `true`
  for 80 s while no session ring existed and the DM node gave 0 bytes. The real
  gate is a timestamped ring under `/data/vendor/slog/`.
- **`stop dmd` does nothing; `stop DM-daemon` works but uses up the boot's
  logging session.** dmd comes back, and no ring is written until a reboot.
- **SCAT cannot drive the DM node.** Its 20-byte `CONTROL_START` reaches the far end
  intact and the modem does not answer, whether dmd runs or not. dmd starts the
  session through `liboemservice.so` (`COMMAND_SET_DM_MODE`).
- **USB DM is blocked by the build type, not by the gadget.** `dm.gs7` exists in
  configfs, but the USB gadget HAL's `getVendorFunctions()` sees
  `ro.build.type=user` and returns before it reads `persist.vendor.usb.usbradio.config`.
  `scat -u` stays untested until a `userdebug` build is available.
- **GSMTAPv3.** SCAT emits some NR frames as GSMTAPv3, which Wireshark 4.7.2 cannot
  dissect. Normal-session NR RRC arrives as v2 and dissects correctly.

## Open questions

Tracked in [ticket 04](../.scratch/live-analyzer/issues/04-node-tap-as-live-path.md)
and in the "Fog" section of the map:

- Can the DM node tap replace the ring path as the default? It is better on every
  measurement so far, but it has run for minutes, not hours, and it has not been
  tested through a modem crash or an airplane cycle.
- Can the DM session be started without dmd, over the same RIL path? That would
  remove the one-session-per-boot limit.
- What is in an NR-only `reconfigurationWithSync` (not nested in LTE), and what
  do handovers from a moving UE look like? Neither has been captured yet.
