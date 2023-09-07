# shocpp

A rudimentary implementation of OCPP, the Open Charge Point Protocol, Version 1.6

## Status

Testing: Daily use backed up by the charger's built-in RFID counters.

## Motivation

- One user of the charger is handicapped and unable to quickly present RFID and plug the cable in one go.
- Spaß am Gerät.

## Goal

- Develop an understanding of OCPP.
- Build a customizable personal OCPP service for internal accounting.

## Scope

- Allow charging initiated by (RF)ID tag presentation (Active *Authorize/StartTransaction* requests from the charger).
- Allow charging initiated by the service (*RemoteStartTransaction* request to the charger with an (RF)ID tag to be assumed as presented).
- Keep track of power consumption on a per-account basis. (RF)ID tags are assigned to accounts.

## Architecture

- *shocpp-backend* implements the entire logic and communicates via stdin/stdout, as known from inetd/xinetd/cgi-bin.
- *shocpp-listener* wraps *shocpp-backend* behind *websocketd*.
- *shocpp-command* passes calls and payloads to *shocpp-backend*, to be sent as requests to the charger.
- *shocpp-caller* is essentially similar to *shocpp-command*, but I use this in a webserver/cgi-bin scenario to invoke hard-coded "canned calls" with random IDs from Siri shortcuts. 
- *shocpp-summary* sums up all transactions (german language, sorry) from the discrete accounting records saved by *shocpp-backend*.

## Limitations

- No authentication of charger, but can be added on a reverse proxy level.
- No encryption, but can be added on a reverse proxy level.
- No tracking of charging duration. Would need to save service-side state for that.
- Known obstacles to operation with more than one charger:
  - *shocpp-command* has no facility to route a request to a specific charger/shocpp process.

## Requirements

- Scripts are written in **Bash**, tested in Version > 5.1 only.
- **websocketd** - for the networking side.
- **jq** - for all JSON operations.

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

## *shocpp-command*

Synopsis:

```shell
shocpp-command <Call> <JSON Payload (the "inner" JSON) for request>
```

Examples:

```shell
bin/shocpp-command RemoteStartTransaction '{"idTag":"00000000"}'
bin/shocpp-command TriggerMessage '{"requestedMessage": "StatusNotification"}'
bin/shocpp-command Reset '{"type":"Soft"}'`
bin/shocpp-command Reset '{"type":"Hard"}'`
```

*shocpp-command* saves the complete request to *run/cmd.json* and signals *SIGUSR1* to the running *shocpp-backend*. *shocpp-backend* traps *USR1* and interrupts all work to send the contents of *run/cmd.json* as a request to the charger. The confirmation from the charger is saved to *run/resp.json*. Meanwhile, *shocpp-command* waits for *run/resp.json* to change and delivers it back as its own output.

## Implemented OCPP requests/calls

### Incoming

Incoming requests are handled by the main *while read* loop in *shocpp-backend* as they arrive on stdin.

- **BootNotification** - Confirmation *status* is always *Accepted*, contains the *currentTime* and *heartbeat* interval.
- **StatusNotification** - Confirmation always contains an empty payload field.
- **Heartbeat** - Confirmation contains the *currentTime* field.
- **Authorize** - The main loop extracts the *idTag* field early on, and the confirmation to *Authorize* sends that result in the *status* field.
- **StartTransaction** - The confirmation to *StartTransaction* returns the same authorization *status* field as with *Authorize*, and also a *transactionId*, generated according to the model described below.
- **StopTransaction** - On receipt of the *StopTransaction* Request, *shocpp-backend* disassembls the *transactionID* and determines account and consumption.

### Outgoing

- The script generates no outgoing requests at this time, other than those handed to it via *shocpp-command*.

## Transaction ID model

*shocpp* keeps no state, but offloads state into the Transaction ID:

- *Rumor has it* that transaction IDs are signed Int32, so max transaction ID is *2.147.483.647*.
- The account ID from *tags.json* for a given (RF)ID tag is multiplied x 10000000. The Wh meter reading at start is converted to kWh and added to it. This number is the Transaction ID.
- On end of transaction, the transaction ID transmitted by the charger in the *StopTransaction* request is disassembled back into account ID and meter reading at start, based on which the consumption is calculated.
- By this calculation, 
  - shocpp can serve up to 214 accounts, and infinite™ (RF)ID tags, and
  - at a meter reading of 10 GWh, the world comes to an end.

## Accounting

Accounting records are written to *accountingdir/year/month/unixtime-random.json*:

```text
$ tree accounting/
accounting/
└── 2023
    └── 09
        ├── 1693673985-cb298fc7f419.json
        ├── 1693688702-6e9c73294697.json
        └── 1693727530-0326fd891def.json

2 directories, 3 files
$ jq . accounting/2023/09/1693727530-0326fd891def.json
{
  "localtime": "2023-09-03 09:52:10",
  "unixtime": "1693727530",
  "account": "family-y",
  "begin_kwh": "9042",
  "end_kwh": "9075",
  "consumed_kwh": "33"
}
```

All records can be summed up using *shocpp-summary* (german language only), e.g. on the first day of a month:

```shell
$ bin/shocpp-summary $(date -d '1 week ago' '+%Y') $(date -d '1 week ago' '+%m')
```

## Future TODOs

- Implement more functionality:
  - Power level, phase switching and overall solar integration, should be easy to implement through *shocpp-command*.
- Command routing in *shocpp-command*:
  - Add identification of charger (serial number) to *shocpp-command* invocation, signal all processes, have the request sent by the process that is in contact with the given charger. 
  - Note that the serial number is never actively transmitted after *BootNotification* and will have to be enumerated (how?) or kept as state.

## Notes

### HOWTO sniff OCPP traffic

Sourced from: https://osqa-ask.wireshark.org/questions/60725/how-to-dump-websockets-live-with-tshark/

```shell
tshark -p -i any -s0 -f 'port 8080' -Y websocket.payload -E occurrence=l -T fields -e ip.src -e ip.dst -e text
```

### Websocket message format

The format of the outer OCPP payload is outlined in the occp-j specification PDF.

### OCPP observations on go-eCharger Gemini

- OCPP compliance
  - *TriggerMessage* request for *MeterValues* supported since FW 055.7 Beta.
- Locally known RFID tags
  - 055.5: Sends locally known RFID tags to OCPP for authorization. Once authorized, consumption is reported to OCPP (*StartTransaction/StopTransaction*) and also added to the local RFID slot. The consumption after *RemoteStartTransaction* for the same RFID tag's ID is also added to the RFID slot.
  - 055.7 BETA: Accepts locally known RFID tags without any OCPP interaction, but adds consumption to the next free (unassigned) RFID configuration slot. Seems erratic. Github Issue: https://github.com/goecharger/go-eCharger-API-v2/issues/176
- Reconnect and reboot behaviour
  - If the charger reconnects after loss of the websocket connection, it checks back in with a *BootNotification* request.
  - If the charger ends a transaction while the websocket connection is not available, it submits the *StopTransaction* request after reconnecting.
  - If the charger ends a transaction and reboots while the websocket connection is not available, and the websocket only becomes available after complete reboot, it submits the *StopTransaction* request after reconnecting.
  - More test cases? Seems solid so far.

## Further reading

- The OCPP specification: https://www.openchargealliance.org/downloads/

## LICENSE

ISC License

**Copyright 2023, the shocpp developers**

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
