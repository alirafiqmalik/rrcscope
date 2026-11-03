# Security and responsible use

## Scope of the tool

rrcscope reads the diagnostic log of **your own** phone's modem. It is passive:
it does not transmit, it does not write modem NV/EFS, it does not send AT or OEM-hook
commands, and it does not flash firmware. The only device state it changes is the
modem logging properties (restored on exit) and, only when you ask for it,
airplane mode.

Use it only on devices and SIM subscriptions that you own or are authorized to
test. Laws on intercepting radio communications differ by country. This tool reads
your own handset's log, not other people's traffic, but you are responsible for
following the law where you are.

## Captures are personal data

A raw `.sdm` or `.pcap` from this tool contains permanent identifiers (IMSI, IMEISV),
temporary identities (M-TMSI, GUTI), your location (cell ID, TAC), and SMS content.

- `.gitignore` excludes `*.sdm`, `*.pcap`, `*.pcapng`, `*.qmdl`, and `captures/`.
  Keep it that way.
- Before you share a finding, mask identifiers. Keep only the IMEI's 8-digit TAC,
  and share tshark summaries, not dumps.
- Do not attach captures to issues.

## Reporting a vulnerability

If you find a bug in this repository that could expose a user's identifiers (for
example a script that leaves data on the device or on a shared host), report it
privately through **GitHub → Security → Report a vulnerability** on this
repository, or contact the maintainer listed in [CITATION.cff](CITATION.cff).

Findings about baseband or network behaviour (for example identifiers sent before
security) belong to the modem vendor or the operator, through their own disclosure
programs.
