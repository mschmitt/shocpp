# tinyocpp

A rudimentary implementation of OCPP, the Open Charge Point Protocol, Version 1.6

## Status

Experimental: Still switching off OCPP to fall back to the charger's internal RFID accounting when I'm done testing.

## Goal

Provide a customizable, reasonably self-hostable OCPP service that ignores the high-scalability and security of existing implementations.

## Scope

- Allow charging initiated by (RF)ID tag presentation (Active *Authorize/StartTransaction* calls from the charger).
- Allow charging initiated by the service (*RemoteStartTransaction* call to the charger with and (RF)ID tag to be assumed as presented).
- Keep track of power consumption on a per-account basis. (RF)ID tags are assigned to accounts.

## Limitations

- No authentication of charger, but can be added on a reverse proxy level.
- No encryption, but can be added on a reverse proxy level.
- Operation with more than one charger was never tested, but should be possible due to the stateless operation model.

## Architecture

- *tinyocpp-backend* implements the entire logic and communicates via stdin/stdout, as known from inetd/xinetd/cgi-bin.
- *tinyocpp-listener* wraps *tinyocpp-backend* behind *websocketd*.
- *tinyocpp-command* passes payloads to *tinyocpp-backend*, to be sent to the charger.

## Configuration

- Listen address via the *WS_ADDRESS* environment (default: *0.0.0.0*, listen on all interfaces).
- Listen port via the *WS_PORT* environment (default: *8080*).
- Path to *tags.json* via *TINYOCPP_TAGSFILE* environment (default: *conf/tags.json,* parallel to tinyocpp's *bin/* directory)
- Path to Accounting directory via *TINYOCPP_ACCOUNTINGDIR* environment (default: *accounting/*, parallel to tinyocpp's *bin/* directory)

## tags.json

*tags.json* is a list of accounts, to which ID tags for the charger are assigned.

```json
[
  {
    "index": 1,
    "account": "company-x",
    "ids": [
      "1111",
      "2222"
    ]
  },
  {
    "index": 2,
    "account": "family-y",
    "ids": [
      "3333",
      "4444"
    ]
  },
  {
    "index": 3,
    "account": "company-z",
    "ids": [
      "5555",
      "6666",
      "7777",
      "8888",
      "9999"
    ]
  }
]
```

## Transaction ID model

*tinyocpp* keeps no state, but offloads state into the Transaction ID:

- *Rumor has it* that transaction IDs are signed Int32, so max transaction ID is *2.147.483.647*.
- The account ID from *tags.json* for a given (RF)ID tag is multiplied x 10000000. The meter reading at start is added to it. This number is the Transaction ID.
- On end of transaction, the transaction ID transmitted by the charger is disassembled back into account ID and meter reading at start, based on which the consumption is calculated.
- Yes, tinyocpp can serve only up to 214 accounts, but many more (RF)ID tags.

## *tinyocpp-command* examples

Synopsis:

`tinyocpp-command <Call> <JSON Payload (the "inner" JSON) for call>`

Examples:

```
bin/tinyocpp-command "RemoteStartTransaction" '{"idTag":"00000000"}'
bin/tinyocpp-command "Reset" '{"type":"Soft"}'`
bin/tinyocpp-command "Reset" '{"type":"Hard"}'`
```

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
