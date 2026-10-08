# OCPP Integration Research: WEG WEMOB Wall

Oct 8, 2026 · @aisac

## Summary

Your WEMOB Wall speaks only OCPP 1.6J, so your API needs a 1.6J Central System (CSMS) that the charger dials into over a WebSocket. Build it as a small OCPP gateway behind your API, with a version-neutral session model so OCPP 2.0.1 chargers can join later.

- **The charger is the client.** It opens `wss://<your-host>/<path>/<ChargeBoxId>` and keeps it open; every command you send travels down that one socket.
- **WEG documents the contract.** Its [OCPP 1.6J application manual](https://static.weg.net/medias/downloadcenter/he3/h26/WEG-WEMOB-OCPP-application-manual-and-configuration-keys-10012744786-pt.pdf) (rev. 01, Feb 2026) lists every supported key and its default. Treat it as your spec for this charger.
- **It is known to interoperate.** WEMOB G2 is listed as working with the open-source [SteVe](https://github.com/steve-community/steve/wiki/Charging-Station-Compatibility) CSMS, and WEG's connectivity guide uses a SteVe-style endpoint as its example server URL.
- **Security tops out at Profile 2.** WEG documents Basic Auth (`AuthorizationKey`) and `wss://` URLs. Client certificates (Profile 3) are not documented.
- **Remote starts skip authorization.** `AuthorizeRemoteTxRequests` is read-only `false`, so your server must validate the idTag when StartTransaction arrives.
- **Energy may start late.** `RandomDelayToCharge` defaults to 600 s on Wall/Parking. Set it to 0 unless you want randomized starts.
- **Recommended path:** capture real traffic with SteVe in a day, build the gateway on a mature library (Python `ocpp` or Node `ocpp-rpc`), test with simulators, then harden security and add smart charging (see Roadmap).

## The charger at a glance

The model code WEMOB-W-012-W-R-1NAC-UL (item 18015118) decodes as Wall, 12 kW, Wi-Fi, RFID, one NACS AC connector, UL version ([WEG model coding](https://static.weg.net/medias/downloadcenter/had/hb6/WEG-WEMOB-brochure-50118441-en.pdf)). Each part of the datasheet shapes the integration:

| Datasheet item | Value | What it means for your API |
| --- | --- | --- |
| Protocol | OCPP 1.6 JSON | Only 1.6J messages; no 2.0.1 device model or TransactionEvent |
| Connectivity | Wi-Fi only (4G and Ethernet not fitted) | 2.4 GHz 802.11 b/g/n, WPA/WPA2-Personal only ([guide](https://static.weg.net/medias/downloadcenter/h41/h38/WEG-WEMOB-wall-parking-g2-10008515171-guia-de-conectividade-pt.pdf)); expect drops and offline replays |
| Connectors | 1 tethered cable, NACS | Sessions run on connectorId 1; connectorId 0 means the whole station |
| Current | 6 to 50 A per phase | Smart-charging limits live between 6 and 50 A |
| Power and supply | 12 kW max, 100–240 V AC, F+N+T or F+F+T | Single-phase supply; Amps is the natural charging-rate unit |
| Energy metering | Included | meterStart, meterStop and MeterValues in Wh are your billing source |
| User interaction | Automatic, RFID, management software | Maps to WEG's AuthenticationMethod 0, 1 or 2 |
| RFID | Reader included, cards sold separately | 13.56 MHz ISO/IEC 14443 A; the card UID becomes the idTag |
| Indicators | LEDs, no display | Display and pricing-text features do not apply |
| Protections | Overcurrent, overtemperature, EV comms loss, RCD 30 mA AC / 6 mA DC, surge | Surface as StatusNotification error codes (see Core charging flows) |

OCPP itself does not carry the connector type, so NACS (SAE J3400) matters only in your own catalog and in roaming data. WEG's published commissioning guide covers the Wall/Parking G2; confirm your UL/NACS unit uses the same setup page (see Open questions).

## OCPP in 2026: which version and why

Build on 1.6J now because it is the only version your charger speaks, but keep your domain model version-neutral. The Open Charge Alliance (OCA) states that 1.6 application logic is not compatible with 2.1, while 2.0.1 logic carries over into 2.1 ([OCA](https://openchargealliance.org/ocpp-2-1-is-now-available/)).

| Version | Released | Status in Oct 2026 | Relevance to you |
| --- | --- | --- | --- |
| 2.1 | Jan 2025; Edition 2 in Dec 2025 | Published as IEC 63584-210:2025; Edition 2 adds certification profiles and test cases ([OCA](https://openchargealliance.org/new-editions-of-the-ocpp-2-1-and-2-0-1-now-available/), [IEC news](https://openchargealliance.org/ocpp-2-1-edition-1-is-now-officially-published-by-iec-as-iec-63584-210-2025/)) | Adds bidirectional charging, DER control, prepaid and ad-hoc payment. Future only |
| 2.0.1 | Apr 2020; Edition 4 in Dec 2025 | Adopted as IEC 63584:2024 ([Wikipedia](https://en.wikipedia.org/wiki/Open_Charge_Point_Protocol)); required for NEVI-funded US chargers ([Maine DOT](https://www.maine.gov/mdot/grants/cfi/docs/Statement%20of%20Work.pdf)) | Not on your charger. Design your model so it can be added |
| 1.6 Security Whitepaper Ed. 3 | Edition 3, v1.3, Feb 2022 | Security profiles, certificates, signed firmware, security events | WEMOB documents only its Basic Auth part |
| 1.6J | Oct 2015; Edition 2 Sep 2017, errata to Apr 2025 | Still the version OCA certifies against today ([certificate](https://openchargealliance.org/wp-content/uploads/2026/04/Certificate_F01.OCA_.0016.1175.CS_CHAEVI.pdf)) and the most deployed ([JOINT](https://jointcharging.com/glossary/ocpp-open-charge-point-protocol/)); OCTT 1.6 test set still maintained | What the WEMOB Wall speaks. Build this first |

Two boundaries to keep in mind. OCPP connects charger and CSMS only; sharing chargers with other networks (roaming) uses OCPI, a separate protocol you can add later ([CitrineOS talk](https://fosdem.org/2025/events/attachments/fosdem-2025-5019-citrineos-one-year-of-progress-of-a-charge-station-management-system/slides/237271/FOSDEM202_F3W3Oca.pdf)). And if you ever deploy with US federal NEVI funds, a 1.6J-only charger would not qualify.

## How OCPP 1.6J works on the wire

The charger opens one long-lived WebSocket to your server, and both sides exchange JSON-array RPC frames over it, with at most one unanswered request per direction at a time.

### Connection

- **URL.** The charger appends its identity to the configured URL: configure `wss://ocpp.example.com/ocpp` and it connects to `wss://ocpp.example.com/ocpp/WEMOB-0001`. Route on the last path segment ([SteVe](https://github.com/steve-community/steve/wiki/OCPP-1.6J-Security-Configuration)).
- **Subprotocol.** The charger offers `ocpp1.6` in `Sec-WebSocket-Protocol`. Echo it back, and close connections that offer none.
- **Authentication.** On profiles 1 and 2 the upgrade request carries `Authorization: Basic base64(<ChargeBoxId>:<AuthorizationKey>)`. Reject it before accepting the WebSocket.
- **Keep-alive.** WebSocket ping/pong (the WEMOB pings every 20 s by default) detects dead sockets. OCPP Heartbeat mainly keeps the charger's clock in sync.

### RPC framing

```json
[2, "19223201", "BootNotification", {"chargePointVendor": "WEG", "chargePointModel": "WEMOB-WALL"}]
[3, "19223201", {"status": "Accepted", "currentTime": "2026-10-08T12:41:00Z", "interval": 300}]
[4, "19223202", "NotSupported", "Action not supported", {}]
```

Vendor and model values above are illustrative; capture the real ones in your first test.

- Message type `2` is a CALL, `3` a CALLRESULT, `4` a CALLERROR. The unique id is a string of up to 36 characters that pairs a response with its request ([summary](https://docs.rs/crate/ocpp_rs/0.3.0/source/doc.txt)).
- Send a charger a new CALL only after the previous one is answered or timed out, so queue commands per charger ([details](https://ocpp.hexdocs.pm/ocpp/protocol/endpoint.html)).
- 1.6J error codes: NotImplemented, NotSupported, InternalError, ProtocolError, SecurityError, FormationViolation, PropertyConstraintViolation, OccurenceConstraintViolation (misspelled in the spec itself), TypeConstraintViolation, GenericError.
- Use UTC ISO 8601 timestamps everywhere; OCPP strongly recommends UTC.

### Boot and registration

1. The charger connects and sends BootNotification with vendor, model, serial and firmware.
2. You answer Accepted, Pending or Rejected, plus `interval` (heartbeat seconds) and `currentTime`.
3. While Pending, the charger sends nothing else, but you may read and change its configuration. Use this to provision new units.
4. While Rejected, it stays silent until the interval passes, then boots again ([spec excerpt](https://docs.rs/crate/rust-ocpp/latest/source/src/v1_6/messages/boot_notification.rs)).
5. Once Accepted, it reports StatusNotification per connector and sends Heartbeat every `interval` seconds.

## Message catalog (1.6J)

OCPP 1.6J has 28 operations in six feature profiles. The WEMOB's `SupportedFeatureProfiles` default lists all six, so plan for the full set but ship the MVP rows first. Read that key at boot rather than assuming it.

| Message | Direction | Profile | When to build | Notes |
| --- | --- | --- | --- | --- |
| BootNotification | Charger → CSMS | Core | MVP | Registration, clock sync, heartbeat interval |
| Heartbeat | Charger → CSMS | Core | MVP | Liveness; you return `currentTime` |
| StatusNotification | Charger → CSMS | Core | MVP | Connector status plus errorCode |
| Authorize | Charger → CSMS | Core | MVP | RFID check; answer within a few seconds |
| StartTransaction | Charger → CSMS | Core | MVP | You assign the integer `transactionId` |
| MeterValues | Charger → CSMS | Core | MVP | Periodic energy, power, current, voltage |
| StopTransaction | Charger → CSMS | Core | MVP | Final meter reading and stop reason |
| RemoteStartTransaction | CSMS → charger | Core | MVP | App start; Accepted means it will try, not that energy flows |
| RemoteStopTransaction | CSMS → charger | Core | MVP | Stops by `transactionId` |
| GetConfiguration | CSMS → charger | Core | MVP | WEMOB returns up to 58 keys per call |
| ChangeConfiguration | CSMS → charger | Core | MVP | Accepted, Rejected, RebootRequired or NotSupported |
| Reset | CSMS → charger | Core | MVP | Soft or Hard reboot |
| TriggerMessage | CSMS → charger | Remote Trigger | MVP | Ask for a fresh StatusNotification, MeterValues or Heartbeat now |
| ChangeAvailability | CSMS → charger | Core | Phase 2 | Operative or Inoperative, for maintenance |
| ClearCache | CSMS → charger | Core | Phase 2 | Empties the authorization cache |
| DataTransfer | Both | Core | Phase 2 | Vendor extensions; WEG uses vendorId `weg` |
| SetChargingProfile | CSMS → charger | Smart Charging | Phase 2 | Current or power limits and schedules |
| ClearChargingProfile | CSMS → charger | Smart Charging | Phase 2 | Removes profiles by id or purpose |
| GetCompositeSchedule | CSMS → charger | Smart Charging | Phase 2 | Shows the limit the charger actually enforces |
| SendLocalList | CSMS → charger | Local Auth List | Phase 2 | WEMOB: 100 entries per call, 1,000 total |
| GetLocalListVersion | CSMS → charger | Local Auth List | Phase 2 | Detects drift between your list and the charger's |
| UpdateFirmware | CSMS → charger | Firmware Mgmt | Phase 2 | Points the charger at WEG's firmware file |
| FirmwareStatusNotification | Charger → CSMS | Firmware Mgmt | Phase 2 | Downloading, Installed, failures |
| GetDiagnostics | CSMS → charger | Firmware Mgmt | Phase 2 | Charger uploads logs to a URL you provide |
| DiagnosticsStatusNotification | Charger → CSMS | Firmware Mgmt | Phase 2 | Upload progress |
| ReserveNow | CSMS → charger | Reservation | Optional | Holds the connector for one idTag |
| CancelReservation | CSMS → charger | Reservation | Optional | Releases a reservation |
| UnlockConnector | CSMS → charger | Core | Skip | Not listed in WEG's manual; the cable is tethered |

WEG's manual describes only the Core messages in detail ([manual](https://static.weg.net/medias/downloadcenter/he3/h26/WEG-WEMOB-OCPP-application-manual-and-configuration-keys-10012744786-pt.pdf)), so test each Phase 2 message against the real unit before relying on it.

## Core charging flows

Treat the StartTransaction and StopTransaction the charger sends as the source of truth for every session. Your commands only ask the charger to act.

### App-initiated session (your main API flow)

&#91;embedded content: App-initiated session · app, backend and charger, StartTransaction highlighted\]

The highlighted StartTransaction is where your API learns that charging began; replies to MeterValues, RemoteStopTransaction and StopTransaction are left out for clarity.

1. The app calls your API to start. Check the charger is connected and the connector is Available or Preparing, then create a pending session.
2. Send `RemoteStartTransaction {connectorId: 1, idTag: <user token>}`. Use a per-user virtual idTag of at most 20 characters.
3. The charger answers Accepted or Rejected. Accepted only means it will try.
4. On the WEMOB, `AuthorizeRemoteTxRequests` is read-only `false`, so it starts without sending Authorize ([manual](https://static.weg.net/medias/downloadcenter/he3/h26/WEG-WEMOB-OCPP-application-manual-and-configuration-keys-10012744786-pt.pdf)).
5. When the EV is plugged in, the charger sends StartTransaction with your idTag. Match it to the pending session and return a `transactionId` with status Accepted.
6. No StartTransaction within `ConnectionTimeOut` (60 s default) plus a margin means the attempt failed. The connector returns to Available.
7. To stop, send `RemoteStopTransaction {transactionId}`. The session ends when StopTransaction arrives.

If StartTransaction carries an idTag you refuse, answer `Invalid`. With `StopTransactionOnInvalidId` true (the default), the WEMOB then ends the session with reason DeAuthorized.

### RFID session (AuthenticationMethod 2, Authorized by OCPP Server)

1. The driver taps a card. The charger sends `Authorize {idTag}`; you answer Accepted, Blocked, Expired, Invalid or ConcurrentTx.
2. The driver plugs in within `ConnectionTimeOut`. StartTransaction arrives with connectorId, idTag, meterStart (Wh) and timestamp; you return a transactionId.
3. While charging, MeterValues arrive every `MeterValueSampleInterval` (15 s default), tagged with the transactionId.
4. A card tap, the EV or a remote stop ends it. StopTransaction carries meterStop, timestamp and reason; energy is meterStop minus meterStart.

With `LocalPreAuthorize` and the authorization cache on (both default true), known cards can start without an Authorize call. StartTransaction is then your first chance to refuse.

### Offline and retries

- While offline, the charger keeps transaction messages and replays them on reconnect. StartTransaction can arrive late with an old timestamp.
- The WEMOB retries a failed transaction message 5 times, waiting 2 s times the number of previous attempts (`TransactionMessageAttempts`, `TransactionMessageRetryInterval`).
- Make StartTransaction idempotent: a retry with the same charger, connector, idTag, timestamp and meterStart gets the same transactionId.
- Never reject StopTransaction or MeterValues. Store first, reconcile later; a server error makes the charger resend the same message ([example](https://github.com/dallmann-consulting/OCPP.Core/issues/31)).
- Offline authorization follows `LocalAuthorizeOffline` (true), `AllowOfflineTxForUnknownId` (false) and WEG's `AlwaysAllowOfflineTransactions` (false, so no free charging while offline).

### Connector status

| Status | Meaning | Suggested API state |
| --- | --- | --- |
| Available | Free for a new user | available |
| Preparing | Cable in or card presented, no transaction yet | preparing |
| Charging | Energy flowing | charging |
| SuspendedEVSE | Charger withholds energy, e.g. a smart-charging limit | paused\_by\_charger |
| SuspendedEV | Vehicle draws no energy, e.g. full or on its own schedule | paused\_by\_vehicle |
| Finishing | Transaction stopped, cable still connected | finishing |
| Reserved | A reservation holds the connector | reserved |
| Unavailable | Taken out of service, e.g. by ChangeAvailability | unavailable |
| Faulted | Error; no energy can flow | faulted |

ConnectorId 0, the whole station, only reports Available, Unavailable or Faulted ([enum docs](https://docs.rs/rust-ocpp/0.2.0/rust_ocpp/v1_6/types/enum.ChargePointStatus.html)).

### Error codes

| Datasheet protection | Likely 1.6 errorCode |
| --- | --- |
| Overcurrent | OverCurrentFailure |
| Overtemperature | HighTemperature |
| EV communication failure | EVCommunicationError |
| Residual current (30 mA AC / 6 mA DC) | GroundFailure |
| Surge or overvoltage | OverVoltage |
| RFID reader | ReaderFailure |
| Energy meter | PowerMeterFailure |
| Weak Wi-Fi | WeakSignal |

This mapping is the usual convention, not confirmed by WEG. Also store the `info` and `vendorErrorCode` fields, where vendors put their own codes.

## Security

Run the WEMOB on Security Profile 2: TLS (`wss://`) plus HTTP Basic Auth with a per-charger key. It is the strongest profile WEG documents for this charger.

| Profile | Transport | How the charger authenticates | Use |
| --- | --- | --- | --- |
| 0 | `ws://`, unencrypted | None | Lab only |
| 1 | `ws://`, unencrypted | HTTP Basic Auth | Trusted network or VPN only |
| 2 | `wss://`, TLS 1.2+ | HTTP Basic Auth; charger verifies your server certificate | Production, recommended |
| 3 | `wss://`, TLS 1.2+ | Client certificate (mutual TLS) | Not documented for the WEMOB |

Profile definitions follow the 1.6 Security Whitepaper Edition 3 as summarized by [SteVe](https://github.com/steve-community/steve/wiki/OCPP-1.6J-Security-Configuration).

### What the WEMOB supports

- `AuthorizationKey`: write-only Basic Auth password, a hex string of 16–20 bytes (32–40 hex digits). WEG recommends generating it randomly.
- `WebManagementEndpoint`: accepts a `wss://` URL (WEG's example is `wss://ocpp.weg.net/ocpp/chargebox`). An invalid URL disconnects the charger until it is re-commissioned on site.
- No `SecurityProfile` key, certificate messages or SecurityEventNotification appear in WEG's manual, so assume they are absent.

### Setup that works

1. Pre-register each ChargeBoxId; refuse unknown ids during the WebSocket handshake.
2. Commission the charger with your `wss://` URL, using a certificate from a widely trusted public CA. Ask WEG which root CAs the firmware trusts; some chargers ship without newer roots ([example](https://github.com/steve-community/steve/wiki/Charging-Station-Compatibility)).
3. On first boot, generate a random 20-byte key and push it with `ChangeConfiguration(AuthorizationKey, <40 hex digits>)`. Store only a hash of it.
4. Require Basic Auth on every later connection. Rotate the key the same way.

One interop trap: the whitepaper treats the key as raw bytes, so the Basic password may be 20 binary bytes rather than the 40-character hex text ([worked example](https://www.winccoa.com/documentation/WinCCOA/latest/en_US/ocpp/ocpp_security.html), [library fix](https://github.com/ChargeTimeEU/Java-OCA-OCPP/pull/143)). Some chargers send the hex text instead ([WARP](https://docs.warp-charger.com/en/docs/interfaces/ocpp)). Decode the header as bytes and accept either form until you see what the WEMOB sends.

### Other hardening

- Close an older socket when the same ChargeBoxId reconnects, and rate-limit failed handshakes.
- Log every raw frame with direction and receive time; it is your audit trail and debugger.
- `EnableSystemHealth` (default false) sends station telemetry to WEG. Decide it under your privacy policy.

## Smart charging

The WEMOB accepts standard 1.6 charging profiles in Amps or Watts: up to 30 installed profiles, stack levels up to 10 and 10 periods per schedule. That covers current caps, load balancing and time-of-use schedules.

### The three profile purposes

- **ChargePointMaxProfile** caps the whole station and can only be set on connectorId 0.
- **TxDefaultProfile** applies to new sessions. On connectorId 0 it covers all connectors; on 1 it covers that connector only.
- **TxProfile** limits one running session. It needs an active transaction on a connector above 0 and overrides TxDefaultProfile while that session lasts. It can also ride inside RemoteStartTransaction.

Within a purpose, the highest stack level in effect wins, and the station never exceeds its ChargePointMaxProfile ([spec text in the library](https://raw.githubusercontent.com/mobilityhouse/ocpp/master/ocpp/v16/enums.py)).

### Example: evening cap, Mexico City time

```json
[2, "sc-001", "SetChargingProfile", {
  "connectorId": 1,
  "csChargingProfiles": {
    "chargingProfileId": 10,
    "stackLevel": 0,
    "chargingProfilePurpose": "TxDefaultProfile",
    "chargingProfileKind": "Recurring",
    "recurrencyKind": "Daily",
    "chargingSchedule": {
      "startSchedule": "2026-10-09T06:00:00Z",
      "duration": 86400,
      "chargingRateUnit": "A",
      "chargingSchedulePeriod": [
        {"startPeriod": 0, "limit": 50.0},
        {"startPeriod": 64800, "limit": 16.0},
        {"startPeriod": 79200, "limit": 50.0}
      ]
    }
  }
}]
```

The schedule starts at local midnight (06:00 UTC, since Mexico City is UTC−6 with no daylight saving). It allows 50 A all day except 16 A from 18:00 to 22:00 local.

### Rules for this charger

- Keep limits between 6 and 50 A, the datasheet range. Pilot-signal charging cannot offer less than 6 A, so expect a lower limit to pause charging (SuspendedEVSE) rather than trickle. Verify on the unit.
- Use `GetCompositeSchedule` to confirm the limit the charger is actually enforcing.
- Track installed profiles in your database and clear them when done; the WEMOB holds at most 30.
- For multi-charger sites, WEG sells the WEMOB Smart Charging System, a local controller that acts as a transparent OCPP proxy ([WEG brochure](https://static.weg.net/medias/downloadcenter/hfc/h57/WEG-WEMOB-folheto-50129799-pt.pdf)). The alternative is balancing centrally with ChargePointMaxProfile and TxDefaultProfile from your CSMS.

## WEMOB setup and configuration

Point the charger at your server once on its local setup page, then manage everything else over OCPP with the keys in WEG's [1.6J application manual](https://static.weg.net/medias/downloadcenter/he3/h26/WEG-WEMOB-OCPP-application-manual-and-configuration-keys-10012744786-pt.pdf) (rev. 01, 16 Feb 2026).

### Commissioning

Steps follow WEG's [connectivity guide](https://static.weg.net/medias/downloadcenter/h41/h38/WEG-WEMOB-wall-parking-g2-10008515171-guia-de-conectividade-pt.pdf) for Wall/Parking G2.

1. Power the station. For 10 minutes it broadcasts a Wi-Fi access point named `WEG-EVSE-xxx`; join it with the password given in WEG's guide.
2. Open http://setup.com or http://10.10.10.1.
3. Under OCPP Config, set:
   - Charge Box ID: case-sensitive; letters, digits, `_` and `-` only.
   - OCPP Server URL: your base URL, e.g. `wss://ocpp.example.com/ocpp`.
   - Charging Authorization: Authorized by OCPP Server.
4. Select your 2.4 GHz network and enter its password.
5. Watch your server for the WebSocket upgrade and BootNotification.

WEG's own example, `ws://wemob-ws.app.wnology.io:80/steve/websocket/CentralSystemService`, ends without an id, which fits the station appending its ChargeBoxId. Confirm the exact path in your logs.

### Standard keys: suggested values

| Key | WEG default | Suggested | Why |
| --- | --- | --- | --- |
| HeartbeatInterval | 3000 s | 300 s, returned as BootNotification `interval` | Faster liveness and clock sync |
| WebSocketPingInterval | 20 s | 20–30 s | Detects dead sockets; keep proxy idle timeouts longer |
| MeterValueSampleInterval | 15 s | 30–60 s | Five measurands every 15 s is chatty on Wi-Fi |
| MeterValuesSampledData | Voltage, Current.Import, Power.Active.Import, Energy.Active.Import.Register, SoC | Energy.Active.Import.Register, Power.Active.Import, Current.Import | SoC is DC-only |
| ClockAlignedDataInterval | 0 (off) | 900 s | Quarter-hour readings for reports, aligned to UTC |
| MeterValuesAlignedData | Same five | Energy.Active.Import.Register | The register is enough for interval reports |
| ConnectionTimeOut | 60 s | 60–120 s | Time to plug in after authorization |
| TransactionMessageRetryInterval | 2 s | 30 s | Spreads 5 retries over about 5 min instead of about 20 s |
| AuthorizationCacheEnabled | true | true | Fast starts for known cards |
| LocalPreAuthorize | true | true, or false for strict control | False forces an Authorize round-trip every time |
| LocalAuthorizeOffline | true | true | Known cards keep working offline |
| AllowOfflineTxForUnknownId | false | false | Avoids unbilled sessions |
| StopTransactionOnInvalidId | true | true | Lets you end sessions you refuse |
| LocalAuthListEnabled | false | true if you sync cards | Up to 1,000 ids, 100 per SendLocalList |
| AuthorizeRemoteTxRequests | false, read-only | Fixed | Remote starts skip Authorize |

### WEG custom keys that matter on the Wall

| Key | Default | Effect | Suggestion |
| --- | --- | --- | --- |
| AuthenticationMethod | 0 | 0 Always Authorized, 1 Local List, 2 OCPP Server | 2 for an API-driven product |
| RandomDelayToCharge | 600 s | Random delay of up to this many seconds before energy flows | 0 unless you want staggered starts |
| FreeModeCustomIdTag | Empty | idTag used in Always Authorized mode; default is `<ChargeBoxId>-<connector>` | Set it if you offer free charging but still track sessions |
| AlwaysAllowOfflineTransactions | false | Free charging whenever the CSMS is unreachable | A business decision |
| AllowChargeAfterBrownOut | false | Resumes a session after a power cut without re-authorization | Consider true for home and fleet |
| ChargeBoxId | Set at commissioning | 4–25 characters; reconnects at least 20 s after a change | Update your registry before changing it |
| WebManagementEndpoint | Set at commissioning | Server URL, 25–255 characters | Test the new endpoint first; a bad URL needs a site visit |
| CustomIdleFeeAfterStop | true | Keeps a transaction open until unplug for idle fees, tied to California Pricing | Verify when StopTransaction is sent on your firmware |
| EnableSystemHealth | false | Sends telemetry to WEG | Your privacy call |

Ignore the display and Station-only keys (Language, ScreenSaverTime, SilentMode, EnableHmiConfiguration, EnableDualConnectorSameEV, HistoricalDataHmiAccess, CustomDisplayCostAndPrice, DefaultPrice). AutoChargeEnabled uses a vehicle id from DC protocols, so it does not apply to this AC unit.

### Vendor messages and firmware

- `DataTransfer {"vendorId": "weg", "messageId": "TriggerMasterTagRegister"}` reopens master RFID card registration; read-only `CheckMasterTag` shows whether one exists. The manual also spells it TriggerTagMasterRegister, so test both.
- Answer unknown inbound DataTransfer with status UnknownVendorId or UnknownMessageId, not a CALLERROR.
- Update firmware with UpdateFirmware, pointing `location` at your model's folder on WEG's server (http://updates.weg.net/chargingstation). Downloads take 3–10 minutes, downgrades are impossible, and the wrong model's folder can damage the station.

### Setup checklist

- [ ] Switch from the factory default, Always Authorized (free charging), to AuthenticationMethod 2
- [ ] Set RandomDelayToCharge to 0, or tell users about the delay of up to 10 minutes
- [ ] Validate idTags in StartTransaction, since remote starts skip Authorize
- [ ] Use a 2.4 GHz WPA2-Personal network with a strong signal at the wall
- [ ] Finish setup inside the 10-minute access-point window, or power-cycle
- [ ] Answer every charger message within a few seconds; the station waits about 10 s (RegularMessageRetryInterval)
- [ ] Dump all keys with GetConfiguration on first boot and store them
- [ ] Know the SW1 reset button: 3 s clears setup and the master card, 5 s also clears the local card list

## Architecture for your API

Put a small stateful OCPP gateway between the chargers and your stateless API, joined by a message bus, so WebSocket concerns never leak into business logic.

&#91;embedded content: Integration architecture · 7 components, the OCPP gateway highlighted\]

Each charger keeps one socket to a gateway node; everything your apps see flows through the stateless charging service and its database.

### Build or adopt

| Option | What you get | Trade-offs | Best when |
| --- | --- | --- | --- |
| Own gateway on a library: Python [ocpp](https://github.com/mobilityhouse/ocpp) (MIT) or Node [ocpp-rpc](https://github.com/mikuso/ocpp-rpc) (MIT) | Full control; events flow straight into your stack | You own edge cases and conformance testing | You are building a product API. Recommended |
| [SteVe](https://github.com/steve-community/steve) (Java, GPL-3.0) as the OCPP layer | Proven with WEMOB G2; [REST API](https://raw.githubusercontent.com/steve-community/steve/master/api-docs.json) for operations, transactions and tags; 1.6 security extensions | No event push in its published API; needs JDK 25 and MySQL or MariaDB; GPL terms apply if you modify and distribute | A fast pilot, or a reference to compare behavior |
| CitrineOS (TypeScript, Apache-2.0) | API-first CSMS, OCPP 2.0.1 certified ([talk](https://fosdem.org/2025/events/attachments/fosdem-2025-5019-citrineos-one-year-of-progress-of-a-charge-station-management-system/slides/237271/FOSDEM202_F3W3Oca.pdf)); 1.6 support since v1.6.0 ([news](https://tfir.io/lf-energy-boosts-open-source-ev-infrastructure-with-citrineos-1-6-0-update/)) | A larger platform to run; its 1.6 support is younger | You expect 2.0.1 chargers soon |
| Hosted OCPP API, e.g. [eDRV](https://docs.edrv.io/docs/set-charging-profiles) or [Plugchoice](https://developer.plugchoice.com/api-reference/charger-actions/set-charging-profile.md) | REST control of chargers without running a CSMS | Recurring fees, less control, lock-in | Time-to-market beats control |
| WEG's WEMOB Management Platform | WEG's optional cloud platform and apps | Third-party API access is unconfirmed | Ask WEG |

### Components

- **OCPP gateway** (stateful): terminates WebSockets, authenticates chargers, validates frames, keeps one command queue per charger, logs raw frames.
- **Message bus** (Redis Streams, NATS, RabbitMQ or Kafka): carries charger events up and commands down, keyed by ChargeBoxId.
- **Charging service** (your existing API, stateless): sessions, users and tags, pricing and rules. It owns the database.
- **Database** (e.g. PostgreSQL): chargers, connectors, sessions, meter samples, tags, commands, raw frames.
- **App channel**: webhooks or a push socket to your apps.

Charger messages that need a decision, Authorize and StartTransaction, use request–reply over the bus with a 2–3 s budget. Everything else is acknowledged at once and processed as an event.

### Data model

| Entity | Key fields | Filled from |
| --- | --- | --- |
| Charger | charge\_box\_id (key), vendor, model, serial, firmware, registration status, auth key hash, last\_seen\_at, gateway node | BootNotification, Heartbeat |
| Connector | charger + connector\_id, status, error\_code, vendor\_error\_code, status\_at | StatusNotification |
| IdToken | id\_tag (20 chars max), user, status, expiry | Your users; RFID card UIDs |
| Session | public UUID, OCPP transaction\_id (integer), charger, connector, id\_tag, start and stop time, meter\_start\_wh, meter\_stop\_wh, stop\_reason, state | Start and StopTransaction |
| MeterSample | session, timestamp, measurand, phase, value, unit, context | MeterValues |
| Command | id, charger, action, payload, status, response, requester, timestamps | Your API, via the gateway |
| OcppFrame | charger, direction, raw JSON, received\_at | Every frame; keep 30–90 days |

Expose your own session UUID. Keep the OCPP integer `transactionId` internal and globally unique, e.g. from a database sequence.

### API surface

| Endpoint | Purpose | OCPP behind it |
| --- | --- | --- |
| `GET /v1/chargers/{id}` | Connectivity, status, last heartbeat | Cached Boot, Status and Heartbeat data |
| `POST /v1/chargers/{id}/sessions` | Start charging for a user | RemoteStartTransaction, then wait for StartTransaction |
| `POST /v1/sessions/{id}/stop` | Stop charging | RemoteStopTransaction |
| `GET /v1/sessions/{id}` | Energy, power, duration, cost | Start/StopTransaction and MeterValues |
| `PUT /v1/chargers/{id}/limit` | Set a current cap or schedule | SetChargingProfile |
| `POST /v1/chargers/{id}/refresh` | Pull fresh status or meter values | TriggerMessage |
| `POST /v1/chargers/{id}/reset` | Reboot | Reset |
| `GET`, `PATCH /v1/chargers/{id}/config` | Read or change keys | GetConfiguration, ChangeConfiguration |
| `PUT /v1/tags/{idTag}` | Manage RFID cards and app tokens | Database, then SendLocalList |
| `POST /v1/chargers/{id}/firmware` | Update firmware | UpdateFirmware |

Commands return 202 with a command id; the outcome arrives by webhook or `GET /v1/commands/{id}`. Accept an `Idempotency-Key` header on every POST. Emit these events: charger.connected, charger.disconnected, connector.status\_changed, session.started, session.meter\_updated, session.stopped, command.completed, charger.faulted.

### Reliability rules

1. Answer charger requests within 2–3 s; do slow work after replying.
2. Keep one in-flight CALL per charger, queue the rest, and time each out at around 30 s.
3. Make StartTransaction idempotent; never reject StopTransaction or MeterValues.
4. Store the charger's timestamps and your receive time, both in UTC.
5. On reconnect, replace the old socket, then send TriggerMessage StatusNotification to refresh state.
6. Validate frames against the official 1.6 JSON schemas, but log rather than reject minor vendor deviations at first.
7. Alert on stuck sessions: Charging status with no MeterValues for several intervals.

### Scaling past one node

- A charger stays on one gateway node for the life of its socket. Record `ChargeBoxId → node` in Redis, or have each node subscribe to command topics for its own chargers ([pattern](https://codibly.com/blog/ocpp-server-implementation)).
- Gateways are I/O-bound, so scale on connection count. One team reports over 10,000 concurrent chargers behind an AWS Network Load Balancer ([Klika Tech](https://careers.klika-tech.com/blog/bridging-the-ev-charging-divide-architecting-a-scalable-ocpp-integration-on-aws/)).
- Use a load balancer that supports long-lived WebSockets, with idle timeouts above the ping interval.
- Drain nodes on deploy by closing their sockets; chargers reconnect on their own.

## Minimal working example (Python)

This prototype runs a 1.6J Central System and one REST endpoint in a single process, using the MIT-licensed [ocpp](https://github.com/mobilityhouse/ocpp) library. It is enough to connect your WEMOB and start a session from an HTTP call. It passed a full simulated session (boot, authorization, remote start, meter values, stop, heartbeat) on ocpp 2.1.0 and websockets 17.2 on 8 Oct 2026. Handler names follow the library's [v1.6 example](https://raw.githubusercontent.com/mobilityhouse/ocpp/master/examples/v16/central_system.py).

```python
# csms.py: prototype OCPP 1.6J Central System plus a REST endpoint
import itertools
import logging
from contextlib import asynccontextmanager
from datetime import datetime, timezone

import websockets
from fastapi import FastAPI, HTTPException
from ocpp.routing import on
from ocpp.v16 import ChargePoint as BaseChargePoint
from ocpp.v16 import call, call_result
from ocpp.v16.enums import Action, AuthorizationStatus, RegistrationStatus

logging.basicConfig(level=logging.INFO)

KNOWN_CHARGERS = {"WEMOB-0001"}               # pre-registered ChargeBox IDs
VALID_TAGS = {"APP-USER-0001", "04A1B2C3D4"}  # app tokens and RFID UIDs
CONNECTED: dict[str, "ChargePoint"] = {}
tx_ids = itertools.count(1)                   # use a database sequence in production


def utc_now() -> str:
    return datetime.now(timezone.utc).isoformat()


def tag_status(id_tag: str) -> AuthorizationStatus:
    return AuthorizationStatus.accepted if id_tag in VALID_TAGS else AuthorizationStatus.invalid


class ChargePoint(BaseChargePoint):
    @on(Action.boot_notification)
    async def on_boot(self, charge_point_vendor, charge_point_model, **kwargs):
        logging.info("%s booted: %s %s %s", self.id, charge_point_vendor, charge_point_model, kwargs)
        return call_result.BootNotification(
            current_time=utc_now(), interval=300, status=RegistrationStatus.accepted
        )

    @on(Action.heartbeat)
    async def on_heartbeat(self):
        return call_result.Heartbeat(current_time=utc_now())

    @on(Action.status_notification)
    async def on_status(self, connector_id, error_code, status, **kwargs):
        logging.info("%s connector %s is %s (%s)", self.id, connector_id, status, error_code)
        return call_result.StatusNotification()

    @on(Action.authorize)
    async def on_authorize(self, id_tag):
        return call_result.Authorize(id_tag_info={"status": tag_status(id_tag)})

    @on(Action.start_transaction)
    async def on_start(self, connector_id, id_tag, meter_start, timestamp, **kwargs):
        tx_id = next(tx_ids)  # production: idempotent insert that returns the same id on retry
        logging.info("%s tx %s started on connector %s by %s at %s Wh",
                     self.id, tx_id, connector_id, id_tag, meter_start)
        return call_result.StartTransaction(
            transaction_id=tx_id, id_tag_info={"status": tag_status(id_tag)}
        )

    @on(Action.meter_values)
    async def on_meter_values(self, connector_id, meter_value, **kwargs):
        return call_result.MeterValues()

    @on(Action.stop_transaction)
    async def on_stop(self, meter_stop, timestamp, transaction_id, **kwargs):
        logging.info("%s tx %s stopped at %s Wh, reason %s",
                     self.id, transaction_id, meter_stop, kwargs.get("reason"))
        return call_result.StopTransaction()


async def on_connect(websocket):
    charge_point_id = websocket.request.path.rstrip("/").split("/")[-1]  # last path segment
    if not websocket.subprotocol or charge_point_id not in KNOWN_CHARGERS:
        return await websocket.close()
    charge_point = ChargePoint(charge_point_id, websocket)
    CONNECTED[charge_point_id] = charge_point
    try:
        await charge_point.start()
    except websockets.ConnectionClosed:
        logging.info("%s disconnected", charge_point_id)
    finally:
        if CONNECTED.get(charge_point_id) is charge_point:
            del CONNECTED[charge_point_id]


@asynccontextmanager
async def lifespan(app: FastAPI):
    server = await websockets.serve(on_connect, "0.0.0.0", 9000, subprotocols=["ocpp1.6"])
    yield
    server.close()
    await server.wait_closed()


app = FastAPI(lifespan=lifespan)


@app.post("/v1/chargers/{charger_id}/remote-start")
async def remote_start(charger_id: str, id_tag: str, connector_id: int = 1):
    charge_point = CONNECTED.get(charger_id)
    if charge_point is None:
        raise HTTPException(status_code=409, detail="Charger is offline")
    result = await charge_point.call(
        call.RemoteStartTransaction(id_tag=id_tag, connector_id=connector_id)
    )
    if result is None:  # the charger answered with a CALLERROR
        raise HTTPException(status_code=502, detail="Charger returned an error")
    return {"status": result.status}  # Accepted means "will try", not "charging"
```

Run it and start a session:

```bash
pip install ocpp websockets fastapi uvicorn
uvicorn csms:app --host 0.0.0.0 --port 8000
# On the charger: Charge Box ID = WEMOB-0001, OCPP Server URL = ws://<server-ip>:9000/ocpp
# (the /ocpp path keeps the URL above WEG's 25-character minimum; the station appends its ID)
curl -X POST "http://localhost:8000/v1/chargers/WEMOB-0001/remote-start?id_tag=APP-USER-0001"
```

Before production:

- Persist chargers, sessions and meter samples, and make StartTransaction idempotent.
- Check Basic Auth during the handshake and serve `wss://` behind TLS.
- Record each command with a timeout so the API can return 202 and report the outcome later.
- Split the gateway from the API when you add a second node.

## Testing and certification

Test in three layers: simulators in CI for every flow, the real WEMOB for vendor behavior, and OCA's test tool only if you need to prove conformance.

| Tool | What it does | Terms | Use it for |
| --- | --- | --- | --- |
| [SAP charging stations simulator](https://github.com/SAP/e-mobility-charging-stations-simulator) | Simulates and scales fleets of OCPP-J 1.6 and 2.0.x stations in Node.js | Open source | Load tests and CI flows |
| [Virtual Charge Point](https://solidstudio.io/products/virtual-charge-point/) | Scriptable command-line charger for 1.6 and 2.0.1 | Apache-2.0 | Scripted scenarios |
| [OCTT](https://openchargealliance.org/test-tool) | OCA's compliance tool; plays the charger against your CSMS using predefined test cases | One-time test-set license plus yearly subscription; 14-day trial with a limited test set ([OCA](https://openchargealliance.org/whats-new-in-the-past-octt-release-octt-release_2026-02/)) | Pre-certification checks |
| [SteVe](https://github.com/steve-community/steve) | Reference CSMS | GPL-3.0 | Capturing real WEMOB traffic, comparing behavior |
| `ocpp` library client | Write your own test charger in Python | MIT | Unit and integration tests |

OCA certification matters if you sell your CSMS. For an in-house backend it is optional, though OCTT runs catch spec mistakes early. A SteVe-based backend, Powerfill, holds OCA certification for 1.6 Core, Smart Charging and Advanced Security ([SteVe README](https://github.com/steve-community/steve)).

### Test plan for the real charger

- [ ] Boot paths: Accepted, Pending (configure while pending) and Rejected
- [ ] Socket drop mid-session; old socket replaced; state refreshed with TriggerMessage
- [ ] RFID session with Accepted, Invalid and Blocked cards
- [ ] Remote start with no plug-in afterwards (ConnectionTimeOut), and with the EV already plugged in
- [ ] Stops by remote command, by card and from the vehicle
- [ ] Wi-Fi cut mid-charge, then restored: queued MeterValues and StopTransaction arrive
- [ ] A retried StartTransaction gets the same transactionId
- [ ] CSMS restart during a session
- [ ] Charging limits of 0, 6, 16 and 50 A; GetCompositeSchedule matches
- [ ] Power cut while charging, with AllowChargeAfterBrownOut false and true
- [ ] Firmware update on a spare unit
- [ ] Basic Auth: wrong key rejected, key rotation, hex text versus raw-byte password

## Roadmap

Four phases take you from a first connection to a production fleet. Each ends in a gate you can check on the real charger.

1. **Phase 0, discovery.** Run SteVe with `docker compose up -d`, point the WEMOB at it, and record a full session, a reboot and an offline period. Dump every key with GetConfiguration.
   - Gate: real BootNotification values, a key dump and frame logs for every MVP message.
2. **Phase 1, MVP gateway.** Build the gateway and the MVP rows of the message catalog. Expose sessions, remote start and stop, and charger status in your API, with a raw frame log.
   - Gate: the boot, session, offline and duplicate items of the test plan pass on the real unit and in a simulator.
3. **Phase 2, production hardening.** Add `wss://` with Basic Auth and key rotation, smart-charging limits, local list sync, firmware updates, monitoring, alerts and webhooks.
   - Gate: a soak test with no lost sessions and meter totals that reconcile.
4. **Phase 3, scale and future.** Split gateway and API nodes behind a load balancer. Add an OCPP 2.0.1 adapter for future chargers, OCPI if you need roaming, and an OCTT run if you will sell the platform.
   - Gate: a load test at your target fleet size with the SAP simulator.

## Open questions for WEG

Ask WEG support these before you lock production behavior; each answer changes code or setup.

- [ ] Does the UL/NACS Wall (item 18015118) use the same setup page as the Wall G2, and can a Basic Auth password be entered there?
- [ ] Which firmware is current for this model, and what is its UpdateFirmware location?
- [ ] Which root CAs does the firmware trust for `wss://`, and can new ones be added?
- [ ] Is the Basic Auth password sent as raw bytes or as the hex text of AuthorizationKey?
- [ ] Is Security Profile 3, or any other 1.6 security-whitepaper message, supported or planned?
- [ ] Is an OCPP 2.0.1 firmware planned for the Wall?
- [ ] Is the unit OCA-certified for 1.6? No WEG certificate turned up in this research.
- [ ] Which errorCode and vendorErrorCode does each protection fault report?
- [ ] Which connector status shows during RandomDelayToCharge, and is 600 s also the default on UL units?
- [ ] Does CustomIdleFeeAfterStop delay StopTransaction until unplug when the CSMS does not implement California Pricing?
- [ ] Which spelling does the master-tag DataTransfer messageId use?
- [ ] Does the WEMOB Management Platform offer a third-party API?

## Sources

The pages behind the facts above, grouped by publisher type. Your attached datasheet (WEMOB-W-012-W-R-1NAC-UL, item 18015118) is the source for the charger specifications.

**WEG**

- [WEMOB OCPP 1.6J Application Manual and Configuration Keys, doc 10012744786 rev. 01](https://static.weg.net/medias/downloadcenter/he3/h26/WEG-WEMOB-OCPP-application-manual-and-configuration-keys-10012744786-pt.pdf)
- [WEMOB Connectivity Guide, Wall and Parking G2, doc 10008515171](https://static.weg.net/medias/downloadcenter/h41/h38/WEG-WEMOB-wall-parking-g2-10008515171-guia-de-conectividade-pt.pdf)
- [WEMOB brochure with model coding, 50118441](https://static.weg.net/medias/downloadcenter/had/hb6/WEG-WEMOB-brochure-50118441-en.pdf)
- [WEMOB Smart Charging System leaflet, 50129799](https://static.weg.net/medias/downloadcenter/hfc/h57/WEG-WEMOB-folheto-50129799-pt.pdf)

**OCA and standards**

- [OCPP 2.1 is now available (OCA)](https://openchargealliance.org/ocpp-2-1-is-now-available/)
- [New editions of OCPP 2.1 and 2.0.1 (OCA)](https://openchargealliance.org/new-editions-of-the-ocpp-2-1-and-2-0-1-now-available/)
- [OCPP 2.1 published as IEC 63584-210:2025 (OCA)](https://openchargealliance.org/ocpp-2-1-edition-1-is-now-officially-published-by-iec-as-iec-63584-210-2025/)
- [OCPP Compliance Test Tool (OCA)](https://openchargealliance.org/test-tool) and [OCTT release 2026-02](https://openchargealliance.org/whats-new-in-the-past-octt-release-octt-release_2026-02/)
- [OCPP 1.6 certificate F01.OCA.0016.1175.CS, Dec 2025 (OCA)](https://openchargealliance.org/wp-content/uploads/2026/04/Certificate_F01.OCA_.0016.1175.CS_CHAEVI.pdf)
- [Open Charge Point Protocol (Wikipedia)](https://en.wikipedia.org/wiki/Open_Charge_Point_Protocol)
- [OCPP version history (JOINT)](https://jointcharging.com/glossary/ocpp-open-charge-point-protocol/)
- [Recharge Maine statement of work, NEVI OCPP requirement](https://www.maine.gov/mdot/grants/cfi/docs/Statement%20of%20Work.pdf)

**Open-source projects**

- [SteVe](https://github.com/steve-community/steve), its [compatibility list](https://github.com/steve-community/steve/wiki/Charging-Station-Compatibility), [1.6J security configuration](https://github.com/steve-community/steve/wiki/OCPP-1.6J-Security-Configuration) and [REST API spec](https://raw.githubusercontent.com/steve-community/steve/master/api-docs.json)
- [mobilityhouse/ocpp](https://github.com/mobilityhouse/ocpp), its [v1.6 central system example](https://raw.githubusercontent.com/mobilityhouse/ocpp/master/examples/v16/central_system.py) and [v1.6 enums](https://raw.githubusercontent.com/mobilityhouse/ocpp/master/ocpp/v16/enums.py)
- [ocpp-rpc](https://github.com/mikuso/ocpp-rpc)
- CitrineOS: [FOSDEM 2025 talk](https://fosdem.org/2025/events/attachments/fosdem-2025-5019-citrineos-one-year-of-progress-of-a-charge-station-management-system/slides/237271/FOSDEM202_F3W3Oca.pdf) and [1.6.0 release news](https://tfir.io/lf-energy-boosts-open-source-ev-infrastructure-with-citrineos-1-6-0-update/)
- [SAP e-mobility charging stations simulator](https://github.com/SAP/e-mobility-charging-stations-simulator) and [Solidstudio Virtual Charge Point](https://solidstudio.io/products/virtual-charge-point/)

**Protocol details and implementation notes**

- [OCPP-J 1.6 summary (ocpp\_rs)](https://docs.rs/crate/ocpp_rs/0.3.0/source/doc.txt), [endpoint rules (ocpp for Gleam)](https://ocpp.hexdocs.pm/ocpp/protocol/endpoint.html), [BootNotification excerpt](https://docs.rs/crate/rust-ocpp/latest/source/src/v1_6/messages/boot_notification.rs), [ChargePointStatus](https://docs.rs/rust-ocpp/0.2.0/rust_ocpp/v1_6/types/enum.ChargePointStatus.html)
- [WinCC OA OCPP security profiles](https://www.winccoa.com/documentation/WinCCOA/latest/en_US/ocpp/ocpp_security.html), [Java-OCA-OCPP binary key fix](https://github.com/ChargeTimeEU/Java-OCA-OCPP/pull/143), [WARP charger OCPP docs](https://docs.warp-charger.com/en/docs/interfaces/ocpp)
- [OCPP.Core issue on a rejected StopTransaction](https://github.com/dallmann-consulting/OCPP.Core/issues/31)
- [OCPP server scaling guide (Codibly)](https://codibly.com/blog/ocpp-server-implementation) and [scalable OCPP on AWS (Klika Tech)](https://careers.klika-tech.com/blog/bridging-the-ev-charging-divide-architecting-a-scalable-ocpp-integration-on-aws/)
- [eDRV charging profiles](https://docs.edrv.io/docs/set-charging-profiles) and [Plugchoice SetChargingProfile API](https://developer.plugchoice.com/api-reference/charger-actions/set-charging-profile.md)
