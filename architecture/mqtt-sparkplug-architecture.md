# Ignition MQTT / Sparkplug B Architecture: Context Summary

Context for an AI assistant helping on this system. Use it to answer "where should this go?" questions. Confirm anything marked *verify* before relying on it.

## 1. Platform

- Ignition 8.1.45; Cirrus Link MQTT Transmission, Engine and Distributor 4.0.14 (older than Cirrus Link's current docs, so paths can differ from them)
- Perspective for HMI; scripting in Jython 2.7
- About 30 sites; three environments: DEV, STG, PROD
- Separate from the ODIN platform; don't describe it in ODIN terms
- No Tag Historian and no MQTT Transmission history store (store and forward). None planned. The only SQL writes are alarm journals
- PROD and STG use Active Directory; DEV does not
- No gateway redundancy

## 2. Gateways

| Role | Count | What runs there |
|---|---|---|
| **Data Collector (DC)** | One per site | PLC device connections (drivers), OPC server connections (can be more than one, e.g. KEPServerEX), MQTT Transmission with one transmitter per environment, remote alarm journal |
| **TagIO** | One per environment, shared by all sites | MQTT Distributor (the broker), MQTT Engine, one tag provider per site (reference tags), alarm journals, a DB connection per site |
| **Frontend (FE)** | One per environment | Perspective HMI; reads TagIO and the DC through remote tag providers; connects directly to the site DBs |

Example names (Orlando):

| Environment | DC | TagIO | FE |
|---|---|---|---|
| DEV | ARG-AB-ORLD-DC | ARG-COR-DEV-TG | ARG-COR-DEV-FE |
| STG | *verify* | ARG-COR-STG-TG | ARG-COR-STG-FE |
| PROD | *verify* | *verify* | *verify* |

- **Transmitters:** each site's DC publishes through 3 transmitters, one per environment (ARG-COR-DEV, ARG-COR-STG, ARG-COR-PROD), each feeding that environment's broker/Engine.
- **Sparkplug IDs:** Group ID is `ARG-ABC` everywhere. Edge Node ID is the site name in caps plus `-DC` (e.g. `ORLANDO-DC`).
- **Primary Host ID:** configured (DEV: `ARG-COR-DEV-TG-MQTT-HOST`).
- **Databases:** each environment has its own SQL Server, all with the same DB names. One DB per site, e.g. `ARG_COR_ORLD`; its store-and-forward folder name is lowercase, `arg_cor_orld`.
- **Alarm journals:** named `<Site>_ALM`. The DC's remote journal store-and-forward folder is per environment, e.g. `orlando_alm_dev`.
- **PLCs:** named by cabinet, e.g. CC101, CC102, CCF03A.

## 3. Tag providers

| Gateway | Provider | Contents |
|---|---|---|
| DC | `[<Site>]` (e.g. `[Orlando]`) | Local provider: diagnostics instances (see section 6) |
| DC | `[System]`, `[MQTT Transmission]` | Built in |
| TagIO | `[<Site>]` | Reference tags to `[MQTT Engine]`. Process tags and their alarms live here |
| TagIO | `[System]`, `[MQTT Engine]` | Built in |
| FE | `[<Site>]` (remote) | Points at TagIO's `[<Site>]` |
| FE | `[<Site>DC]` (remote) | Points at the DC's `[<Site>]` |

The FE has only those two remote providers per site. Older `<Site>MQTT` and `TagIOSystem` remote providers were removed.

## 4. Data paths

Full diagram of all three environments: [mqtt-sparkplug-flow.md](mqtt-sparkplug-flow.md).

```
PLC ─driver─► DC ─MQTT Transmission (Sparkplug B, TLS)─► TagIO Distributor ─► MQTT Engine
                                                                               │
                                                     [<Site>] reference tags ◄─┘
                                                               │
                                                  GAN remote provider [<Site>]
                                                               ▼
                                                              FE  ◄── JDBC ── SQL Server (site DB)

DC ─GAN: remote alarm journal─► TagIO ─JDBC─► SQL (alarm journal)
DC ─GAN: remote provider [<Site>DC]─► FE   (diagnostics and controls only)
```

## 5. Where things go (agreed rules)

1. **Process data takes one path only: MQTT.** The FE reads process values only from `[<Site>]` (TagIO), never from `[<Site>DC]`. Two paths to one value can disagree, bypass TagIO's alarms and stale handling, and load the DC.
2. **Plumbing data stays off MQTT.** Gateway health, device and driver diagnostics, Transmission state, and commands stay on each gateway's own provider and reach the FE over the Gateway Network (GAN). Reason: monitoring must not travel over the thing it monitors. When MQTT breaks, the DC card and its controls must still work.
3. **On the FE, only `Diagnostics*` folders may bind to `[<Site>DC]`.**
4. **Split of work:** MQTT carries bulk process data from many sites into one model on TagIO. GAN carries the Ignition-specific traffic: alarms, diagnostics, message handlers, commands.
5. **Exception, a deliberate canary:** a heartbeat published *through* MQTT, to prove the MQTT path is moving (section 8).

## 6. Site diagnostics design

**One UDT definition per gateway type**, each using only that gateway's local providers. One definition serves every environment; values are filled in per instance. Every folder has a `Status` integer (0 unknown/gray, 1 OK, 2 warn, 3 fault), and parents roll up with `max()` of their children.

| Where | Path | UDT | What it covers |
|---|---|---|---|
| DC | `[<Site>]Diagnostics` | DCDiagnostics | TagProvider (heartbeat = `[System]Gateway/UptimeSeconds`), Gateway resources, MQTTTransmission, RemoteAlarmJournal, GatewayNetwork (to TagIO, to FE), Control, Info |
| DC | `[<Site>]DiagnosticsDevices/<device>` | DeviceDiag | One instance per PLC connection |
| DC | `[<Site>]DiagnosticsOPC/<server>` | OPCDiag | One instance per OPC server connection |
| TagIO | `[<Site>]Diagnostics` | TagIODiagnostics | TagProvider, Gateway, MQTTDistributor, MQTTEngine, EdgeNode (Online, Current Server, Data Latency, births/deaths), AlarmJournal, Database, GatewayNetwork (to FE), Control, Info |

- **FE reads them as** `[<Site>DC]Diagnostics…` and `[<Site>]Diagnostics…`.
- **Heartbeat:** gateway uptime is the heartbeat; no extra polling. A remote reader judges a provider by heartbeat quality, and by whether the value keeps increasing.
- **DeviceDiag uses two layers:**
  - `[System]Gateway/Devices/<dev>` is the anchor; it exists whenever the DC runs.
  - The `Driver/` folder reads the driver's own `[Diagnostics]` node: `ns=1;s=[<dev>][Diagnostics]/…` on the Ignition OPC UA Server. It holds Connected, State, Hostname, Port, a `Load/` folder (throughput ratio, request times) and an `Identity/` folder (product, firmware, serial).
  - Parameters: `DriverDiag` (1/0), `LoadWarnPct`, `RequestWarnMs`.
  - **A driver diagnostic tag that goes bad is a fault, not unknown.**
- **Engineer controls (engineer role only):**
  - `Control/` tags in the UDTs, writing True passes through to the module. DC: RefreshEdgeNode, ResetMetrics, RefreshAll, DiscoverEdgeNodes. TagIO: Rebirth, ResetNodeInfo (*verify paths on 4.0.14*).
  - Device and OPC actions go through a `diagCommand` Gateway Network message handler on the DC: restartDevice, enable/disableDevice, bounceOPC.
  - FE remote providers must allow writes for the Control tags.
- **View:**
  - One card per gateway (DC, TagIO, FE) with sub-cards. The UDT folders mirror the sub-cards.
  - Material Design 3 styling.
  - Built dynamically from Ignition components (not imported SVG) by GitHub Copilot, working from an MD spec. The project has a Tailwind-based Advanced Stylesheet (`stylesheet.css`).

## 7. Bad data, fallback values and alarms

- **No fallback values on the TagIO reference tags.** They are a pure mirror of `[MQTT Engine]`, value and quality. The fallback is *last value + bad quality*. A substituted value looks like real data and corrupts the alarms configured on those tags.
- **If a consumer needs a defined value,** handle it at that consumer: an `isGood()` guard in an expression returning an explicit "unknown", or a check in the binding or script.
- **Alarms on reference tags:** set **Ignore Bad Quality** so stale data doesn't trip or clear them.
- **Root-cause alarms:**
  - One comms alarm per site, from `[<Site>]Diagnostics/EdgeNode/Online` (TagIO).
  - One per device, from DeviceDiag `Status` on the DC ("PLC device disconnected"). It shows on the FE only if alarms are enabled on the `[<Site>DC]` remote provider (*open item*).
- **Perspective:** keep quality overlays on. Custom symbols should gray out on bad quality.

## 8. Telling an MQTT problem from a hardware problem

| TagIO EdgeNode/Online | DC heartbeat (GAN) | DeviceDiag Driver/Connected | Meaning | Owner |
|---|---|---|---|---|
| False | Live | (still readable) | MQTT path: Transmission, broker, network to TagIO | SCADA / IT |
| False | Stale | Unknown | DC gateway or site network down | IT / site |
| True | Live | False | PLC unreachable: power, cable, switch, IP | Field |
| True | Live | True, some tags bad | Tag or config problem | SCADA |

**Values can also freeze with GOOD quality.** Causes: a late death certificate (keepalive timeout), an edge node online but not publishing (Refresh Required, broken subscriptions), or values frozen before MQTT (stalled tag group, PLC not updating). Quality can't catch these, so the detection is proof of motion:

- **Node heartbeat:** DC `[System]Gateway/UptimeSeconds`, published through Transmission and checked for age on TagIO. It proves the DC → MQTT → Engine path is moving.
- **PLC heartbeat:** a counter in each PLC program, published with its tags and checked on TagIO. It proves the path end to end.
- **Reading them together:**
  - Node heartbeat frozen: MQTT path problem.
  - Node moving, one PLC frozen, and the driver connected: PLC stuck or tag group stalled.
  - Node moving, one PLC frozen, and the driver disconnected: hardware or network.
- **Clocks:** timestamps come from the DC's clock, so NTP sync matters, or use generous thresholds.

## 9. Open items

- Whether one DC serves all three environments (if so, it needs per-environment DCDiagnostics instances).
- TagIO Control tag paths (Node Control/Rebirth, Node Info/Reset Info) unverified on Engine 4.0.14.
- Alarms on the `[<Site>]DC` remote provider, so DC-side device alarms reach the FE alarm table.
- FE remote-provider write access for the engineer Control tags.
- Whether device-disable removes the driver's `[Diagnostics]` node (test on DEV).
- DeviceDiag thresholds (`LoadWarnPct` 90, `RequestWarnMs` 500) are starting guesses; tune from observed values.
- Optional `Freshness/` folder in TagIODiagnostics for the node and PLC heartbeats (proposed, not built).

## 10. Verified path gotchas (differ from the docs)

- `[System]Gateway/Devices/<dev>/Status|Enabled` (not `Device`)
- `[System]Gateway/OPC/Connections/<server>/Connected|Enabled|State`
- `[System]Gateway/StoreAndForward/<name>/…` (no spaces)
- Gateway Network folders are prefixed `_0:0:` (e.g. `_0:0:ARG-COR-DEV-FE`)
- Engine: `Edge Nodes/<Group>/<Node>/Node Info/Data Latency` (no "(ms)"), `Online DateTime`
- Transmission: the MQTT Client folder sits under the edge node: `Transmission Info/Transmitters/<tx>/Edge Nodes/<Group>/<Node>/MQTT Client/…`. `Last Refresh` is under `Transmission Control`. There is no `Refresh Alarms` tag.
- Remote tag providers are resolved on the gateway that *reads* them; a reference tag can only use providers on its own gateway.
- UDT parameters of type Float imported wrong; use Integer (e.g. percent) instead.
