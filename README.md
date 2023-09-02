# shocpp

A rudimentary implementation of OCPP, the Open Charge Point Protocol, Version 1.6

## Status

Experimental: Still switching off OCPP to fall back to the charger's internal RFID accounting when I'm done testing.

## Goal

Build a customizable private OCPP service that can safely ignore the public-infrastructure aspects of most other implementations.

## Scope

- Allow charging initiated by (RF)ID tag presentation (Active *Authorize/StartTransaction* calls from the charger).
- Allow charging initiated by the service (*RemoteStartTransaction* call to the charger with an (RF)ID tag to be assumed as presented).
- Keep track of power consumption on a per-account basis. (RF)ID tags are assigned to accounts.

## Limitations

- No authentication of charger, but can be added on a reverse proxy level.
- No encryption, but can be added on a reverse proxy level.
- Known obstacles to operation with more than one charger:
  - *shocpp-command* has no facility to route a command to a specific charger/shocpp process.


## Architecture

- *shocpp-backend* implements the entire logic and communicates via stdin/stdout, as known from inetd/xinetd/cgi-bin.
- *shocpp-listener* wraps *shocpp-backend* behind *websocketd*.
- *shocpp-command* passes payloads to *shocpp-backend*, to be sent to the charger.

## Configuration

- Listen address via the *WS_ADDRESS* environment (default: *0.0.0.0*, listen on all interfaces).
- Listen port via the *WS_PORT* environment (default: *8080*).
- Path to *tags.json* via *SHOCPP_TAGSFILE* environment (default: *conf/tags.json,* parallel to shocpp's *bin/* directory)
- Path to Accounting directory via *SHOCPP_ACCOUNTINGDIR* environment (default: *accounting/*, parallel to shocpp's *bin/* directory)

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

*shocpp* keeps no state, but offloads state into the Transaction ID:

- *Rumor has it* that transaction IDs are signed Int32, so max transaction ID is *2.147.483.647*.
- The account ID from *tags.json* for a given (RF)ID tag is multiplied x 10000000. The Wh meter reading at start is converted to kWh and added to it. This number is the Transaction ID.
- On end of transaction, the transaction ID transmitted by the charger is disassembled back into account ID and meter reading at start, based on which the consumption is calculated.
- By this calculation, 
  - shocpp can serve up to 214 accounts, and infinite™ (RF)ID tags, and
  - at a meter reading of 10 GWh, the world comes to an end.


## *shocpp-command* examples

Synopsis:

```shell
shocpp-command <Call> <JSON Payload (the "inner" JSON) for call>
```

Examples:

```shell
bin/shocpp-command "RemoteStartTransaction" '{"idTag":"00000000"}'
bin/shocpp-command "Reset" '{"type":"Soft"}'`
bin/shocpp-command "Reset" '{"type":"Hard"}'`
```

## HOWTO sniff OCPP traffic

Sourced from: https://osqa-ask.wireshark.org/questions/60725/how-to-dump-websockets-live-with-tshark/

```shell
tshark -p -i any -s0 -f 'port 8080' -Y websocket.payload -E occurrence=l -T fields -e ip.src -e ip.dst -e text
```

## TODO

- Systemd unit, generalize runtime environment.
- error/exit/cleanup behaviour is not quite right. (set -o errexit seems wrong, better ignore some errors and keep going, Ctrl+C, something something)

## Future TODOs

- Implement more functionality:
  - Power level and phase switching, solar integration.
- Command routing in *shocpp-command*:
  - Add identification of charger (serial number) to *shocpp-command* invocation, signal all processes, have the command sent by the process that is in contact with the given charger. Note that the serial is never actively transmitted after *BootNotification* and will have to be kept as state.

## OCPP observations on go-eCharger Gemini

- OCPP compliance
  - *TriggerMessage* call for *MeterValues* supported since FW 055.7 Beta.

- Locally known RFID tags
  - 055.5: Sends locally known RFID tags to OCPP for authorization. Once authorized, consumption is reported to OCPP (*StartTransaction/StopTransaction*) and also added to the local RFID slot. The consumption after *RemoteStartTransaction* for the same RFID tag's ID is also added to the RFID slot.
  - 055.7 BETA: Accepts locally known RFID tags without any OCPP interaction, but adds consumption to the next free (unassigned) RFID configuration slot.

- Reconnect and reboot behaviour
  - If the charger reconnects after loss of the websocket connection, it checks back in with a *BootNotification*.
  - If the charger ends a transaction while the websocket connection is not available, it submits *StopTransaction* after reconnecting.
  - If the charger ends a transaction and reboots while the websocket connection is not available, and the websocket only becomes available after complete reboot, it submits *StopTransaction* after reconnecting.
  - More test cases? Seems solid so far.

## Further reading

- The OCPP specification: https://www.openchargealliance.org/downloads/

## LICENSE

ISC License

**Copyright 2023, the shocpp developers**

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
