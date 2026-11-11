# Huawei E5573: SMS over HTTP with debug AT and separate USSD

## Evidence

Anton Tryapkin supplied `5gmodem-diag-20261108-2058.txt` (normal mode) and `з.txt` (debug mode), stored locally under `/home/do/Downloads/5gmodem-logs/tryapkin/`. Both reports run app 3.9.25, Huawei E5573Bs-320 firmware 21.329.01.00.00, with `kind=hilink` at USB path `1-1` and a responding Huawei API at `192.168.8.1`.

Normal mode reports USB `12d1:14db`, no AT ports, `at_debug=0` and SMS read/send/USSD ports set to the private IP. Debug mode reports `12d1:1566`, six serial ports, `at_debug=1`, a metrics AT port `/dev/ttyUSB0`, and all four global messaging/AT ports rewritten to that serial port. The diagnostic's direct HTTP metrics call still succeeds in debug mode. Its USB enumeration omits the network device, but the API response and the earlier interface-up log establish that HTTP access has not vanished.

The user reports that inbox contents disappear in debug mode, AT USSD works there, and manually selecting HiLink only restores SMS after a software reboot. The reports do not contain the actual failed AT SMS exchange or a raw HTTP inbox response; they therefore do not establish a modem firmware error or prove recovery on hardware.

## Confirmed application defects

`smsbridge.sh` chose HTTP only when `hl_web_metrics` was true or no character AT port existed. E5573 is deliberately an AT metrics device in debug mode, so exposing a valid AT port also switched its SMS backend to serial. A manually selected SMS IP did not affect that decision: the condition consulted the modem's saved metrics AT port instead. This is an incorrect transport decision, not a persistent daemon that merely needs restarting.

The `smsopt` persistence endpoint accepted only device paths, silently refusing a HiLink IP sent by the SMS page. Also, automatic port setup and the combined debug-page AT field synchronize global SMS/USSD values, making an independent global USSD selection unstable.

## Changes

- `hl_sms_web` separates SMS routing from metric transport. Huawei devices explicitly registered as HiLink use the HTTP SMS API in either composition. Existing special web-only profiles retain HTTP; other vendors with working serial ports retain their previous AT default. Explicit HiLink/IP SMS choices are honored without clearing the metrics AT port or rebooting.
- SMS read/send defaults are resolved before backend selection. The same decision covers inbox, sent messages, counts, sending and deletion. CPMS setup also avoids applying serial SMS storage configuration to HTTP-backed Huawei HiLink devices.
- `smsopt` accepts HiLink/valid IPv4 SMS port values only while the active device is registered as HiLink; ordinary serial-path validation remains in place.
- The expandable USSD settings panel has a separate **USSD sending port** field. It stores `ussd_port` in the active modem's own section, separately from global SMS defaults, so resolve/ensureports do not overwrite it. Empty means automatic selection. The USSD worker accepts an override only when it is an existing character device owned by that USB modem; stale or foreign ports fall back to the existing automatic path. The page recognizes the override when checking whether a request can be sent.

No USB composition switch, modem restart or Internet reconnect is required by these changes. Prepared for release 3.9.33; no router deployment or hardware validation has been performed.

## Validation and next field check

Six offline behavior tests cover normal/debug Huawei SMS routing, unchanged non-HiLink and other serial backends, explicit HTTP selection with a live AT port, real SMS dispatch branches, scoped USSD overrides, and persisted port validation. A JavaScript fixture exercises the actual USSD settings callbacks, including two modem identities and clearing the override. Existing Huawei API parsing/session tests and the full issue-regression suite are also run.

After installing a release containing this change, keep the modem in debug mode, keep its metrics AT port, and check SMS read/send/delete plus AT USSD without rebooting. The existing logs are sufficient to implement this fix; request more diagnostics only if that field check fails. Earlier SMS-body/authentication work remains documented in the Huawei modem page and the local `pending-hilink-e5573-and-issue68.md` notes; this report describes a separate backend-selection defect on the updated firmware.
