# MT5700 follow-up notes

Saved at the user's request on 2026-11-08. The [reference comparison](2026-11-08-tdtech-reference.md) retains pinned source links, protocol formats, implemented fixes, local validation and remaining limits. The [original issue investigation](2026-11-07-issue-68.md) retains the earlier evidence and diagnostic requests.

These items are deferred. Do not treat the source comparison or offline checks as validation on hardware.

1. **Temperature display first:** decode the twelve CHIPTEMP fields by documented sensor names/order and decide whether to show all sensors or the useful subset. Preserve the hottest-valid-sensor summary. Units are 0.1 degree Celsius; 65535 means unavailable. Do not display missing sensors as zero. Confirm raw readings and field names before changing the UI.
2. **USB cold boot:** verify configuration selection without waiting for watchdog recovery. Explain the observed change from vendor NCM ff/02/16 to standard CDC NCM. Preserve PCUI and a working network function across repeated coldplug, hotplug and module reboot. Generic cdc_ncm must not be assumed to support the earlier vendor layout.
3. **IPv6 and heating:** confirm the parked modem interface stays down, only the current parent/IPv6 client owns eth4, and the Invalid argument loop stops. Verify CPU/temperature recovery rather than assuming it. Do not disable IPv6 globally to hide the ownership bug.
4. **Bands:** verify LTE and shared NR lock readback after apply, unlock and reboot. The supported-band list currently follows the reference model table, not a live capability query. NSA/SA do not have independent masks. Frequency/cell locks need their own implementation with band, SCS, PCI and mobility semantics.
5. **Additional metrics:** consider SSB beams, MCS tables, LTE transmission mode, more secondary carriers and QoS/data-flow information. MCS RAT flags are not MIMO rank. NR SSB ARFCN can differ from downlink ARFCN. Match secondary measurements by RAT/channel and filter unavailable sentinels. The current card has four secondary-carrier slots.
6. **SMS:** test receive/send, Unicode and multipart messages, SM/ME storage, persistence of CPMS selection and descending batch deletion. Confirm ambiguous-result handling without duplicate submission. IMS/CEUS/SMSC provisioning is a separate settings feature, not an automatic polling action.

The current local changes also fix CID/status confusion in NDISSTATQRY, USB-vs-Ethernet autodial-mode checking, TD Tech HCSQ offsets, hexadecimal LTE/NR PCI, serving/secondary/neighbor measurements and the literal-backslash-r CMGS submission format. Their exact implementation and verification are described in the comparison report.
