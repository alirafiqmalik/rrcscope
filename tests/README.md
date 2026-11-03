# tests/

[← Project README](../README.md)

No test needs a phone. The shell tests put a fake `adb` and a fake SCAT on `PATH`.
The Python tests use the [synthetic fixture](../fixtures/README.md) and inline PDML.

```bash
PYTHONPATH=src python3 -m unittest discover -s tests -v   # 16 tests, standard library only
for t in tests/*.sh; do bash "$t"; done                   # 3 shell suites
```

`tshark` must be on `PATH` for the pcap-backed tests. If it is missing, the
`SyntheticCaptureTests` class is skipped.

| File | Covers |
|---|---|
| [`test_events.py`](test_events.py) | Summary classifier: specific rules win (`…Complete` is not `…Reconfiguration`), paging is dropped, the real LTE attach summary contains attach, identity, and measurements |
| [`test_fields.py`](test_fields.py) | Field analyzer: measId → trigger from an earlier measConfig, remove-then-add in one message, unknown measIds, handover only with `mobilityControlInfo`, every NR serving MO, the LTE/NR conversion ranges |
| [`test_live_tail_ring.sh`](test_live_tail_ring.sh) | Ring follower: follows a rotation, restores logging, reaps only the old ring's tail with a quote-safe pattern, says why it stops when there is no ring, continues after a `stat` that races a rotation |
| [`test_experiment_node.sh`](test_experiment_node.sh) | DM-node tap: refuses to start without a session ring, sends the reader `< /dev/null`, stops the reader before SCAT so the pcap is flushed, reports the byte count, restores all three logging properties |
| [`test_dump_pcap.sh`](test_dump_pcap.sh) | `dump_pcap.sh`: summary, empty pcap, missing path, directory, verbose |

Most shell assertions lock down a failure that once cost a device run and a reboot.
The comment above each scenario tells which failure it prevents.
