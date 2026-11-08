# TD Tech MT5700 comparison and issue #68 follow-up

## Reference

Inspected `LianXia233/luci-app-mt5700` at commit `87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db`:

- [Dialing and NDIS parsing](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/src/rust/src/atclient.rs): CID precedes IPv4 status; autodial includes a data-path mode; both AUTODIAL/AUTODAIL response spellings occur.
- [USB and address setup](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/root/etc/init.d/at-webserver): serial binding and ordinary DHCP/DHCPv6 interfaces. It does not provide a special kernel network driver or configuration-selector fix for the vendor NCM layout.
- [USB mode choices](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/htdocs/luci-static/resources/view/at-webserver/dial.js): SETMODE 4 is Linux NCM, independent of sysfs configuration 1/2.
- [Metrics formats](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/htdocs/luci-static/resources/at-webserver/rpc.js), [secondary cells and lock formats](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/htdocs/luci-static/resources/at-webserver/parse.js), [lock readback](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/htdocs/luci-static/resources/view/at-webserver/network_settings.js).
- [SMS transactions](https://github.com/LianXia233/luci-app-mt5700/blob/87f9f75a89a6be3bd0c41f1c73721b6a2fcba7db/src/rust/src/sms.rs): one-line literal-backslash-r CMGS, explicit submission acknowledgement, reverse-order batch deletion and no retries after ambiguous submission.

These sources establish the other app's implementation and expected formats. They are not proof that every firmware or host kernel supports the same behavior. The changes here implement protocol facts independently; no reference backend or UI is imported.

## Confirmed local defects

| Area | Before | Change |
| --- | --- | --- |
| USB startup | TD Tech serial binding only; no selection of the network configuration | Load cdc_ncm; select configuration 2 only without an existing network function; prepare before fibocom netdev lookup |
| Serial/network binding | Broad ff match also covered vendor NCM; generic migration had no TD Tech subclass filter | Remove temporary USB dynamic ID; release configuration-2 network interfaces; migrate subclass 06 serial interfaces only |
| Ownership | Netdev appearance restarted auto=0/swapped profiles sharing the USB path | Skip disabled/parked interfaces; reject parked fibocom setup before touching hardware |
| NDIS status | CID 1 was interpreted as a connected data plane | Require CID 1 and IPv4 status 1 |
| Autodial | Only the enable bit was read | Accept both response spellings; align explicitly wrong data mode to USB and verify; preserve short replies and manual mode |
| Bands | No MT5700 band profile | Read/write LTE and shared NR band locks; preserve SYSCFGEX fields on mode changes |
| HCSQ | Huawei offsets, missing WCDMA/GSM fields | TD Tech offsets, saturation and unavailable-value filtering |
| Serving identity | Only PCI, treated as decimal | MONSC direct values, hexadecimal LTE/NR PCI, CID/TAC, WCDMA PSC/LAC |
| Secondary metrics | Frequency/bandwidth only | Match MONSSC/CASCELLINFO measurements by RAT and channel |
| Neighbors / power | Not read | MONNC; LTE/GUL TXPOWER total, labeled NR PUSCH power |
| SMS submission | Standard prompt/Ctrl-Z dialog | Dedicated CMGS wire form, explicit acknowledgement, no ambiguous-result fallback/retry |
| SMS deletion | Batch slot order followed UI order | Highest slots first on TD Tech |

The latest [issue comment](https://github.com/fildunsky/luci-app-5gmodem/issues/68#issuecomment-6707400151) and its diagnostic file also show both `modem` and `modem2` up on eth4, despite the old modem profile having auto=0 and modem_swapped=1. Their DHCPv6 children compete on the same device, and modem_6 continuously restarts with Invalid argument. Fixing the netdev hook addresses the demonstrated ownership error, rather than disabling IPv6 globally. The precise failing ubus operation inside the platform's DHCPv6 handler is not captured.

## Remaining limits

- The old descriptor capture has ff/02/16 NCM; the later enumeration has standard CDC NCM and receives an IPv4 lease and IPv6 address. No captured command explains the descriptor change. A configuration-2 write is a bounded startup fix to test, not proof of support for the earlier vendor layout. That layout is logged and left out of serial binding; no untested huawei_cdc_ncm binding or persistent SETMODE write is forced.
- Reference LTE/NR band tables are model presets, not measured supported-band capabilities. NR uses one shared lock for NSA and SA, shown through the single NR row. Separate NSA/SA controls would misrepresent this modem.
- The main app exposes a serving cell and four secondary slots, neighbor measurements, power and the hottest valid temperature. It does not replicate the reference's SSB beam matrix, all twelve sensor readings, MCS/transport tables, data-flow/QoS panels or more than four secondary slots. No MIMO rank or modulation is guessed from an MCS RAT flag. Frequency/cell locks and IMS/CEUS provisioning remain separate features; this change adds band selection, not the reference's full settings UI.
- Existing PDU receive/list/storage handling is retained. Its wire formats are standard, but a mock cannot confirm real incoming SMS, SIM/ME persistence or over-the-air delivery.

## Validation

Deferred follow-up is tracked in [MT5700 follow-up notes](2026-11-08-tdtech-follow-up.md).

The diagnostic collector includes read-only TD Tech mode, autodial, NDIS, lock, serving/secondary/neighbor and SMS-storage queries for the selected port's own USB device, so the next report contains the raw inputs for these parsers.

`tools/issue-regressions/test_tdtech.py` exercises NDIS CID/status, autodial spelling and USB-mode verification, parked-interface rejection, netdev ownership, USB configuration preservation, complete band readback, validated writes, neighbor numeric formats and literal CMGS submission with ambiguous outcomes. Profile smoke checks cover the TD Tech modem and band profiles against success, error, empty, malformed and reference-format scenarios. The prior runtime regression suite remains applicable to the shared SMS path.

The field acceptance test is a cold boot with only the current modem's interface active, a prompt network device/address without waiting for watchdog reboot, actual traffic routed through the modem with wired WAN excluded from the test, stable DHCPv6/CPU/temperature, LTE/NR band readback after applying and unlocking, and receive/send/delete SMS in both available stores. No claim of live validation is made until those checks are returned.
