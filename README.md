# tinyocpp

A rudimentary implementation of OCPP, the Open Charge Point Protocol, Version 1.6

------

## Status

Experimental: Still switching off OCPP to fall back to the charger's internal RFID accounting when I'm done testing.

## Goal

Provide a customizable, reasonably self-hostable OCPP service that ignores the high-scalability complexity of existing implementations.

## Scope

- 1 charger will talk to 1 dedicated *tinyocpp* service.
- Allow charging initiated by RFID token presentation (Active *Authorize/StartTransaction* calls from the charger).
- Allow charging initiated by the service (*RemoteStartTransaction* call to the charger).
- Keep track of power consumption per ID.

## Limitations

- No authentication of charger.
- No encryption.
- Supports only one charger, so no load sharing.
- Scaling out to multiple chargers by running multiple instances on multiple ports should be possible.

## Architecture

- *tinyocpp-backend* implements the entire logic and communicates via stdin/stdout, as known from inetd/xinetd/cgi-bin.
- *tinyocpp-listener* wraps *tinyocpp-backend* behind *websocketd*.
- *tinyocpp-command* passes payloads to *tinyocpp-backend*, to be sent to the charger.

## Configuration

- Listen address via the *WS_ADDRESS* environment (default: empty, listen on all interfaces).
- Listen port via the *WS_PORT* environment (default: *8080*).
- Path to *tags.json* via *TINYOCPP_TAGSFILE* environment (default: *conf/tags.json,* parallel to tinyocpp's *bin/* directory)
- Path to Accounting directory via *TINYOCPP_ACCOUNTINGDIR* environment (default: *accounting/*, parallel to tinyocpp's *bin/* directory)

## *tinyocpp-command* examples

Synopsis:

`tinyocpp-command <Call> <JSON Payload (the "inner" JSON) for call>`

Examples:

`bin/tinyocpp-command "RemoteStartTransaction" '{"idTag":"00000000"}'`
`bin/tinyocpp-command "Reset" '{"type":"Soft"}'`
`bin/tinyocpp-command "Reset" '{"type":"Hard"}'`

## HOWTO sniff OCPP traffic

Sourced from: https://osqa-ask.wireshark.org/questions/60725/how-to-dump-websockets-live-with-tshark/

`tshark -p -i any -s0 -f 'port 8080' -Y websocket.payload -E occurrence=l -T fields -e ip.src -e ip.dst -e text`

## TODO

- Specify and implement accounting including periodic reports.
- Systemd unit.
- Something about the exit/cleanup trap is not quite right. (Ctrl+C, error exits, something something)

## Future TODOs

- Implement more functionality:
  - Collect status data.
  - Power level and phase switching, solar integration.

## Further reading

- The OCPP specification: https://www.openchargealliance.org/downloads/

## LICENSE

ISC License

**Copyright 2023, the tinyocpp developers**

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.