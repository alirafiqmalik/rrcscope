# scripts/

[← Project README](../README.md)

Bash on the host, `sh` on the phone. Each script sources [`lib.sh`](lib.sh) for the
shared helpers (`adb_su`, `scat_cmd`, logging enable/restore). The diagrams of
the live paths are in [docs/architecture.md](../docs/architecture.md).

> [!NOTE]
> Every script that changes the phone puts it back on exit, including after
> Ctrl-C. The scripts do not toggle eSIM profiles, and they toggle airplane mode
> only when you give `--airplane`.

## At a glance

| Script | Needs phone | What it does |
|---|:---:|---|
| [`setup_venv.sh`](setup_venv.sh) | | Creates `.venv` with SCAT (`signalcat`) and `nasrrc` installed in editable mode. Run it once. |
| [`live_tail_ring.sh`](live_tail_ring.sh) | ✔ | **Main live path.** Streams the dmd session ring to SCAT, sends GSMTAP to UDP 4729 until Ctrl-C. |
| [`live_analyze.sh`](live_analyze.sh) | | Prints mobility events from an interface (`lo` by default), a pcap, or a FIFO. |
| [`live_replay_sdm.sh`](live_replay_sdm.sh) | | Replays a saved `.sdm` as live GSMTAP, or with `--analyze` sends it directly into the analyzer. |
| [`capture_nas_rrc.sh`](capture_nas_rrc.sh) | ✔ | Copies the newest ring from the phone to `captures/<timestamp>/cap.sdm`. |
| [`decode_nas_rrc.sh`](decode_nas_rrc.sh) | | Decodes a `.sdm` into a pcap plus summary, verbose, and NAS-only text. Prints a control-plane inventory. |
| [`dump_pcap.sh`](dump_pcap.sh) | | Prints saved pcaps as one tshark line per frame (`-v` for a full dissection). |
| [`experiment_scat_diag.sh`](experiment_scat_diag.sh) | ✔ | Experimental DM-node paths: `probe`, `node` (works), `serial`, `usb`, `restore`. |
| [`lib.sh`](lib.sh) | | Shared helpers. Sourced only, do not run it. |
| [`device/`](device/) | (on phone) | Helpers pushed to `/data/local/tmp` by `experiment_scat_diag.sh`. |

## Typical sessions

**Live, two terminals:**

```bash
./scripts/live_tail_ring.sh                 # terminal 1 — Ctrl-C restores logging
./scripts/live_analyze.sh                   # terminal 2 — mobility events from lo
```

**Offline, from a snapshot:**

```bash
./scripts/capture_nas_rrc.sh captures/run1  # pull cap.sdm
./scripts/decode_nas_rrc.sh captures/run1/cap.sdm
./scripts/live_analyze.sh captures/run1/cap.pcap --kinds
```

**No phone, no capture rights:**

```bash
./scripts/live_replay_sdm.sh --analyze captures/run1/cap.sdm
./scripts/live_analyze.sh fixtures/synthetic-lte/lte-mobility.pcap
```

## Script reference

### `live_tail_ring.sh [--airplane]`

Enables verbose modem logging, finds the newest `sbuff_<timestamp>.sdm`, confirms
that it grows, and follows it across rotations into a FIFO named `*.sdm`, which
SCAT decodes as an always-on log. `--airplane` does one airplane cycle after the
stream starts. Use it to force a full detach and re-attach.

| Variable | Default | Meaning |
|---|---|---|
| `GSMTAP_HOST` / `GSMTAP_PORT` | `127.0.0.1` / `4729` | Where SCAT sends radio 0 |
| `LAYERS` | `nas,rrc` | SCAT layers (`pdcp,rlc,mac,ip` are available too) |
| `FIFO` | `/tmp/nasrrc-live.sdm` | Must contain `.sdm` |
| `POLL_SECS` | `2` | How often to look for a newer ring |
| `DRAIN_SECS` | `2` | Grace period for the old ring before detaching |
| `STALL_POLLS` | `15` | Same-size polls before the script asks dmd to flush |
| `FLUSH_LIMIT` | `3` | Flush requests before giving up (the logger wedges if asked forever) |

### `live_analyze.sh [source] [nasrrc options]`

The source is the first argument that is not an option. A path or a name that
ends in `.pcap` is read as a file (no capture rights needed). Any other name is an
interface, captured through `dumpcap` and filtered to `dst host $GSMTAP_HOST`.
Other options go to `python -m nasrrc`, for example `--json` or `--kinds`.

### `live_replay_sdm.sh [--analyze] <cap.sdm> [layers]`

Without `--analyze`, the script replays the dump to `GSMTAP_HOST:GSMTAP_PORT` for
Wireshark. With `--analyze`, SCAT writes a pcap stream into a FIFO, and the analyzer
reads it with `tshark -r`, so dumpcap is not used.

### `capture_nas_rrc.sh [outdir] [--airplane]` · `decode_nas_rrc.sh <cap.sdm> [layers]`

The snapshot pair. The decode writes `cap.pcap`, `cap.summary.txt`,
`cap.detail.txt`, and `cap.nas.txt` next to the dump. Then it prints the message
inventory, the NAS messages sent before security (security-header type 0), and an
NR frame count.

### `dump_pcap.sh [pcap-or-dir ...] [-v]`

The default target is `captures/`. An empty pcap prints `(empty)` and does not
cause a tshark error.

### `experiment_scat_diag.sh <mode> [--secs N] [--pcap FILE] [--keep-dmd] [--start-magic X]`

| Mode | Changes the phone | Result |
|---|---|---|
| `probe` | no | Shows how dmd starts, who holds `/dev/umts_dm0`, and which USB gadget functions exist |
| `node` | logging properties only, then restored | **Works.** A read-only tap on the DM node next to dmd, about 44 ms median latency |
| `serial` | suspends dmd (with a dead-man restore) | Does not work. The modem does not answer SCAT's `init_diag` |
| `usb` | USB gadget (with a dead-man restore) | Blocked on `user` builds |
| `restore` | puts everything back | Safe to run at any time |

`--secs 0` runs `node` until Ctrl-C. Other variables: `DM_NODE`
(`/dev/umts_dm0`), `LOGGING_WAIT_SECS` (`60`), `START_MAGIC` (`0x41414141`).
If the host dies, run the restore manually:

```bash
adb shell su -c 'sh /data/local/tmp/nasrrc_dm_restore.sh'
```

## `device/` — scripts that run on the phone

| Script | Role |
|---|---|
| [`nasrrc_dm_read.sh`](device/nasrrc_dm_read.sh) | Read-only tap. The node returns 0 bytes when its queue is empty, so the tap runs `cat` again in a loop |
| [`nasrrc_dm_bridge.sh`](device/nasrrc_dm_bridge.sh) | Two-way bridge for the `serial` experiment. The reader handles node → host, and this script adds host → node |
| [`nasrrc_dm_restore.sh`](device/nasrrc_dm_restore.sh) | Undoes everything: reaps the tap and bridge, resets the USB properties, starts `DM-daemon`. Can be armed with a delay as a dead-man switch |

## Conventions

- `scat_python` uses `$SCAT_PY` if it is set, or else `.venv/bin/python`.
- `adb_su` wraps its argument in single quotes. A `pkill -f` pattern sent through
  it must have no spaces and no quotes, or it matches unrelated processes on the
  phone. See `reap_remote_tails` in `live_tail_ring.sh`.
- Raw `.sdm`, `.pcap`, and `.qmdl` files contain subscriber identities (IMSI/TMSI,
  IMEISV, SMS). `captures/` and these extensions are in `.gitignore`.
