# Baseband NAS attach capture — Pixel 7a

[← Research index](../README.md) · [← Project README](../../README.md)

Frozen prior experiment. The maintained capture/decode/live scripts and classifier
live at the repo root ([`scripts/`](../../scripts/), [`src/nasrrc/`](../../src/nasrrc/)).
Handoff that started this repo: [`docs/live-analysis-handoff.md`](../../docs/live-analysis-handoff.md).

Live capture of the cellular modem's LTE Attach/Detach for the active eSIM, decoded
to NAS/RRC. Captured 2026-08-13 on the rooted Pixel 7a (Samsung Exynos Modem 5300,
Shannon/SIPC over PCIe). The [sequence diagram](#sequence-diagram) tags every message with its on-air protection.

## What was captured

An airplane-mode cycle drove a full detach then a full re-attach. SCAT decoded both
from the modem diagnostic stream. The re-attach is a **Combined EPS/IMSI Attach**,
home PLMN Truphone (234/25) roaming on Verizon (311/480), cell PCI 310 / Band 5 /
TAC 16898.

Message flow (relative times):

- t≈36.3s — airplane ON: PDN disconnect, Deactivate EPS bearer, EMM Detach (switch-off).
- t≈37.8s — airplane OFF: MIB/SIB acquisition, RRCConnectionRequest/Setup/SetupComplete
  carrying Attach request + PDN connectivity request.
- t≈37.8s — Identity request/response, then ESM information request/response.
- t≈40.5s — SecurityModeCommand/Complete, UE capability, Attach accept + Activate
  default EPS bearer, Attach complete, EMM information.
- post-attach — an 11-fragment SMS over NAS (CP-DATA/RP-DATA).

## Sequence diagram

Every message carries its on-air protection: `[BCAST]` unauthenticated broadcast,
`[PLAIN]` no integrity and no ciphering, `[INTEG]` integrity only, `[CIPHER]` ciphered.

```mermaid
sequenceDiagram
    autonumber
    participant M as UE modem Shannon g5300q<br/>active eSIM zz_truphone
    participant N as E-UTRAN eNB<br/>PCI 310 Band 5
    participant E as EPC MME<br/>Verizon 311/480 roaming

    Note over M: Capture path — dmd streams /dev/umts_dm0 → .sdm → SCAT (-t sec)<br/>SDM Start metadata leaks device IMEI 35497743••••••• (masked) + IMEI2 + confpack zz_truphone
    Note over M,E: Home PLMN 234/25 (Truphone) roaming on Verizon 311/480. Combined EPS/IMSI attach, GUTI-based.
    Note over M,E: PROTECTION TAG on every message —<br/>[BCAST] unauthenticated broadcast · [PLAIN] NAS/RRC, no integrity + no cipher (spoofable)<br/>[INTEG] integrity-protected, NOT ciphered (readable, tamper-evident) · [CIPHER] ciphered on air<br/>NB: Shannon logs NAS post-decipher, so a captured header-type 0 after security is a log artifact, not the on-air state.

    rect rgba(128,128,128,0.08)
    Note over M,E: Phase 0 — airplane ON forces graceful detach (t≈36.3s)
    M->>E: ESM PDN disconnect request [CIPHER]<br/>PTI, linked EPS bearer id
    E->>M: Deactivate EPS bearer context request [CIPHER]<br/>EPS bearer id, ESM cause = 36 regular deactivation
    M->>E: Deactivate EPS bearer context accept [CIPHER]<br/>EPS bearer id, PTI
    M->>E: EMM Detach request [PLAIN]<br/>detach type = combined EPS/IMSI, switch-off · EPS mobile identity = GUTI · sec-hdr type 0
    end

    rect rgba(70,130,180,0.10)
    Note over M,N: Phase 1 — airplane OFF, cell acquisition (t≈37.77s)
    N-->>M: MIB [BCAST]<br/>BCCH-BCH · SFN=228 · dl-Bandwidth · PHICH cfg
    N-->>M: SIB1 [BCAST]<br/>PLMN-id 311/480 · TAC 16898 · cellIdentity · cellBarred=notBarred · SI scheduling
    N-->>M: SIB2/3/4/5/8/15 [BCAST]<br/>RACH cfg · PRACH · UL power ctrl · resel thresholds · intra/inter-freq neighbours · CDMA2000
    end

    rect rgba(128,128,128,0.08)
    Note over M,E: Phase 2 — RRC connection setup + Attach request piggybacked (pre-AS-security)
    M->>N: RRCConnectionRequest [PLAIN]<br/>UL-CCCH · ue-Identity = S-TMSI (from GUTI) · establishmentCause = mo-Signalling
    N-->>M: RRCConnectionSetup [PLAIN]<br/>DL-CCCH · SRB1 config (RLC AM, logicalChannelConfig)
    M->>N: RRCConnectionSetupComplete [PLAIN]<br/>selectedPLMN-Identity=1 · dedicatedInfoNAS wraps ↓ NAS Attach request [INTEG] ↓
    Note over M,E: NAS Attach request [INTEG] — sec-hdr type 1 (integrity, reused GUTI ctx, NOT ciphered)<br/>EPS attach type = combined EPS/IMSI · old GUTI · UE network capability (EIA/EEA/UEA) · DRX param · ESM msg container →<br/>PDN connectivity request: request type = initial · PDN type = IPv4v6 · ESM info transfer flag = 1 · APN requested via ESM info<br/>(cold attach with no cached NAS context would send this as [PLAIN] type 0)
    N->>E: Initial UE message (S1AP) [CIPHER]<br/>relays NAS Attach request · eNB UE S1AP id · TAI · E-UTRAN CGI
    end

    rect rgba(70,130,180,0.10)
    Note over M,E: Phase 3 — identity solicitation (PLAIN) then ESM info (CIPHER)
    E->>M: EMM Identity request [PLAIN]<br/>identity type = IMEISV · sec-hdr type 0
    M->>E: EMM Identity response [PLAIN]<br/>mobile identity = IMEISV 35497743•••••••• (masked) (device IMEI2 + SVN) · sec-hdr type 0
    Note over M,E: PERMANENT EQUIPMENT ID IN THE CLEAR — full IMEISV over the air, unciphered, before AS security.<br/>tshark: "Plain NAS message, not security protected". This is the IMSI/IMEI-catcher primitive.<br/>No EPS-AKA Authentication req/resp this attach — network reused the GUTI security context.
    E->>M: ESM information request [CIPHER]<br/>PTI · request for APN + PCO behind NAS security
    M->>E: ESM information response [CIPHER]<br/>APN = truphone.com.mnc025.mcc234.gprs · PCO (DNS req, IPCP) — deferred behind security, NOT leaked on air
    end

    rect rgba(128,128,128,0.08)
    Note over M,E: Phase 4 — AS (RRC) security + UE capability (t≈40.46s)
    E->>M: RRC SecurityModeCommand [PLAIN→establishes AS]<br/>securityAlgorithmConfig: cipheringAlgorithm=eea · integrityProtAlgorithm=eia · integrity-protected with K_RRCint
    M->>E: RRC SecurityModeComplete [INTEG]<br/>SRB1 now integrity+cipher active (AS security established)
    Note over M,E: This is the RRC (Access Stratum) SMC — NOT a NAS SMC.<br/>No NAS Security Mode Command ran: the reused GUTI NAS context was already active from before this attach.
    E->>M: UECapabilityEnquiry [CIPHER]<br/>ue-CapabilityRequest = eutra
    M->>E: UECapabilityInformation [CIPHER]<br/>UE-EUTRA-Capability: band combos, CA, category, feature groups
    end

    rect rgba(70,130,180,0.10)
    Note over M,E: Phase 5 — attach accept + default bearer (CIPHER, AS-protected)
    E->>M: RRCConnectionReconfiguration [CIPHER]<br/>radioResourceConfigDedicated · dedicatedInfoNAS wraps ↓ NAS Attach accept ↓
    Note over M,E: NAS Attach accept [CIPHER] — EPS attach result = combined EPS/IMSI · GUTI reallocation ·<br/>TAI list (311/480, TAC 16898) · T3412 periodic-TAU timer · T3402 ·<br/>+ Activate default EPS bearer context request: EPS bearer id · EPS QoS/QCI · APN · PDN address (IPv4v6) · APN-AMBR
    M->>E: RRCConnectionReconfigurationComplete [CIPHER]
    M->>E: NAS Attach complete [CIPHER]<br/>wraps Activate default EPS bearer context accept (EPS bearer id)
    E->>M: EMM information [CIPHER]<br/>full + short network name · UTC time + local time zone + daylight saving
    E->>M: UEInformationRequest-r9 [CIPHER]
    M->>E: UEInformationResponse-r9 [CIPHER]
    end

    rect rgba(128,128,128,0.08)
    Note over M,E: Phase 6 — SMS over NAS (post-attach)
    E->>M: DL NAS transport [CIPHER]<br/>wraps CP-DATA / RP-DATA — 11-fragment concatenated SMS (TP-UD)
    M->>E: UL NAS transport [CIPHER]<br/>wraps CP-ACK / RP-ACK
    Note over M,E: Header-type 0 appears in the log here but this is post-attach → CIPHERED on air.<br/>The type-0 is the Shannon post-decipher log artifact, NOT cleartext. SMS body left in session capture only.
    end
```

## Two IMEI leak points, confirmed

The IMEI leaves the device on two independent channels:

- **Diagnostic stream.** Every SDM Start Response carries the device IMEI
  `35497743•••••••` (TAC shown, serial masked), IMEI2, and the active profile confpack `zz_truphone` in cleartext.
- **Over the air.** The NAS **Identity Response** carries the full **IMEISV**
  (`35497743••••••••`, device IMEI2 + software version) to the network. It is sent
  **before AS security**, unciphered. The network requested it with an Identity request
  of type IMEISV.

This attach reused the GUTI security context, so no EPS-AKA Authentication
request/response ran. The eNB still pulled the IMEISV in the clear.

## Cryptographic protection by message

Every message is tagged in the diagram with its on-air protection. The Shannon modem
logs NAS *after* deciphering, so a captured security-header type of 0 only means
"unprotected" for messages sent before security activates. After security is up, a
logged type-0 is a decode artifact, not the on-air state. The classes below reflect the
on-air state, cross-checked with tshark.

### No cryptographic protection — readable AND spoofable

No integrity, no ciphering. An on-air attacker reads these and a fake base station can
forge or solicit them. This is the pre-authentication surface that makes IMSI/IMEI
catchers and fake-eNB attacks possible.

| Message | Carries | Why unprotected | Attack it enables |
|---|---|---|---|
| MIB | SFN, DL bandwidth, PHICH config | BCCH broadcast, no sender auth | Fake eNB advertises a cell UEs will camp on |
| SIB1 | PLMN 311/480, TAC 16898, cellId, cellBarred | BCCH broadcast | Spoof PLMN/TAC to lure or trigger TAU |
| SIB2/3/4/5/8/15 | RACH, power control, reselection thresholds, neighbours | BCCH broadcast | Manipulate reselection to pull UEs onto a rogue cell |
| RRCConnectionRequest | S-TMSI, establishmentCause | UL-CCCH before AS security | Identity linkage by S-TMSI; connection spoofing |
| RRCConnectionSetup | SRB1 config | DL-CCCH before AS security | Rogue eNB completes the setup handshake |
| RRCConnectionSetupComplete | selectedPLMN, wraps NAS Attach request | sent before AS security | Carrier for the plaintext NAS below |
| EMM Detach request (switch-off) | detach type, GUTI | NAS sec-hdr type 0 (verified) | Forged detach — signalling denial of service |
| EMM Identity request (IMEISV) | identity type = IMEISV | NAS sec-hdr type 0 | Fake MME solicits the permanent equipment id |
| **EMM Identity response** | **full IMEISV `35497743••••••••`** | **NAS sec-hdr type 0, "Plain NAS, not security protected"** | **Harvest the permanent IMEI over the air — the IMEI-catcher primitive** |

The Identity Response is the sharp one: the network asked for the IMEISV and the UE
returned the full permanent equipment identity with zero protection, before AS security,
even though a NAS security context existed. Any active fake eNB can replay the Identity
request(IMEISV) and collect it. This is a second IMEI-exposure channel alongside the RSP
`deviceInfo` leak — one over the cellular air interface, one over the eSIM download.

### Integrity-protected, NOT ciphered — readable, tamper-evident

Contents are in the clear on air but signed, so an attacker reads them and cannot forge
them undetected.

| Message | Carries | Protection |
|---|---|---|
| NAS Attach request | attach type, old GUTI, UE network capability, DRX, PDN connectivity request (PDN type IPv4v6, ESM info flag) | NAS sec-hdr type 1 via reused GUTI context. A cold attach with no cached context sends this as type 0 — fully unprotected. |

The UE network capability (supported EIA/EEA algorithms) travels here in the clear. That
is the field a bidding-down attacker targets on a cold, type-0 attach.

### Ciphered on air — protected

Confidentiality plus integrity. Not readable on air (the log shows plaintext only because
Shannon decodes post-decipher): ESM information request/response (so the APN
`truphone.com.mnc025.mcc234.gprs` is NOT leaked), RRC SecurityModeCommand/Complete,
UE capability, NAS Attach accept + Activate default EPS bearer, Attach complete, EMM
information, and all SMS-over-NAS.

Note the SMC here is the **RRC (Access Stratum)** SecurityModeCommand. No NAS SMC ran —
the reused GUTI NAS context was already active.

## The test eSIMs do not attach

The enabled test profiles (Speedtest Travel, BetterRoaming — Truphone, simless/wildcard)
do not hold a cellular attach. The modem loops in LTE PHY cell search on PCI 310 with
marginal signal (RSRP −83 to −142 dBm) and never completes RRC. Steady-state data rides
IWLAN (Wi-Fi calling); CS/PS on WWAN read `NOT_REG_OR_SEARCHING`. The complete attach in
the capture belongs to the device's working roaming line, seen on the shared modem log.

## Reproduce

Use the repo-root scripts. Live stream until Ctrl-C: [`scripts/live_tail_ring.sh`](../../scripts/live_tail_ring.sh). Optional one-shot reattach: `--airplane`. Do not toggle eSIM profiles.

Modem is Samsung Shannon, not Qualcomm — use the SDM path, not DIAG/QMDL.

```bash
# 1. enable the silent modem logger (root)
adb shell su -c 'setprop persist.vendor.sys.modem.logging.enable true; \
  setprop vendor.modem.logging.shannon_logging true; start modem_logging_start'
# dmd now writes /data/vendor/slog/sbuff_*.sdm

# 2. optional: force a reattach (or pass --airplane to live_tail_ring.sh)
adb shell su -c 'cmd connectivity airplane-mode enable';  sleep 5
adb shell su -c 'cmd connectivity airplane-mode disable';  sleep 30

# 3. pull + decode (SCAT = fgsect/scat, pip name signalcat)
adb shell su -c 'cp /data/vendor/slog/sbuff_<ts>.sdm /data/local/tmp/; chmod 644 /data/local/tmp/*.sdm'
adb pull /data/local/tmp/<file>.sdm .
python3 -m scat.main -t sec -d <file>.sdm -F attach.pcap -L nas,rrc
tshark -r attach.pcap -T fields -e frame.time_relative -e _ws.col.Info

# 4. revert (root)
adb shell su -c 'setprop persist.vendor.sys.modem.logging.enable false; start modem_logging_stop'
```

## Scope and hygiene

Read-only diagnostic capture. No NV/EFS write, no OEM-hook/AT commands, no firmware
flash. Raw `.sdm`/`.pcap` hold real subscriber traffic (M-TMSI, IMEISV, SMS body) and
stay in the session scratchpad — they are not committed. Identifiers here are masked.
Device state was reverted: logger disabled, generated logs removed, both profiles
returned to Disabled.
