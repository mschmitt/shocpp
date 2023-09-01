# tinyocpp

A rudimentary implementation of OCPP, the Open Charge Point Protocol, Version 1.6

------

**Status: Work in Progress**

------

## Goal

Provide a customizable, easily self-hostable OCPP service that ignores the high-scalability complexity of existing implementations.

## Scope

- 1 charger will talk to 1 dedicated *tinyocpp* service.
- Allow charging initiated by RFID token presentation (Active *Authorize/StartTransaction* calls from the charger).
- Allow charging initiated by the service (*RemoteStartTransaction* call to the charger).
- Keep track of power consumption per ID.

## Limitations

- There is no argument against scaling out to multiple chargers by running multiple instances. However, no infrastructure is in place for shared storage of authentication and accounting data, and paths and ports are not configurable so far.

## Architecture

- *tinyocpp-backend* implements the entire logic and communicates via stdin/stdout, as known from inetd/xinetd/cgi-bin.
- *tinyocpp-listener* wraps *tinyocpp-backend* behind *websocketd*.
- *tinyocpp-command* passes payloads to *tinyocpp-backend*, to be sent to the charger.

## *tinyocpp-command* examples

Syntax is: tinyocpp-command &lt;Call&gt; &lt;JSON Payload (the "inner" JSON) for call&gt;

`bin/tinyocpp-command "RemoteStartTransaction" '{"idTag":"00000000"}'`
`bin/tinyocpp-command "Reset" '{"type":"Soft"}'`
`bin/tinyocpp-command "Reset" '{"type":"Hard"}'`

## HOWTO sniff OCPP traffic

Sourced from: https://osqa-ask.wireshark.org/questions/60725/how-to-dump-websockets-live-with-tshark/

`tshark -p -i any -s0 -f 'port 9220' -Y websocket.payload -E occurrence=l -T fields -e ip.src -e ip.dst -e text`

## TODOs before completion

- Implement accounting with storage in file system.
- Move away from Port 9220, which was carried over from the OCPP implementation for ioBroker.
- Introduce configuration file:
  - Listener Port
  - Paths:
    - Allowed tags
    - Accounting data

## Community TODOs

- Implement more functionality:
  - Collect status data (I honestly don't care about Grafana dashboards, never have).
  - Power level and phase switching, solar integration.

## Further reading

- The OCPP specification: https://www.openchargealliance.org/downloads/

## LICENSE

ISC License

**Copyright 2023, the tinyocpp developers**

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.