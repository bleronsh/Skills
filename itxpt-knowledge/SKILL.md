---
type: context
description: >
  ITxPT specification knowledge base for the YES-EU Fleet Management Platform.
  Auto-loaded to provide persistent ITxPT architecture context without re-reading
  the 13 source PDFs each session. Covers S01 (hardware), S02 (onboard services),
  and S03 (back-office) from the "Linden" release.
---

# ITxPT Knowledge Base — YES-EU Fleet Management

## Release and Version Context

- **Current named release**: `Linden`
- All DNS-SD TXT records MUST include `release=Linden`
- Source specs directory: `ITxPT_Doc/` in project root (13 PDFs)

| Spec    | Title                            | Version  |
|---------|----------------------------------|----------|
| S01     | Installation Requirements        | v2.3.2   |
| S02P00  | Networks and Protocols           | v2.2.1   |
| S02P01  | Module Inventory                 | v2.2.0   |
| S02P02  | Time                             | v2.2.1   |
| S02P03  | GNSS Location                    | v2.3.0   |
| S02P04  | FMStoIP                          | v2.3.0   |
| S02P05  | VEHICLEtoIP                      | v2.2.1   |
| S02P06  | AVMS                             | v2.3.0   |
| S02P07  | APC                              | v2.2.2   |
| S02P08  | MADT                             | v3.0.1   |
| S02P09  | MQTT Broker                      | v2.3.1   |
| S03P00  | Back-Office Overview             | v3.0.0   |
| S03P01  | TiGR                             | v3.0.2   |

---

## S02P00 — Networks and Protocols (Foundation)

### Service Discovery (DNS-SD / mDNS) — MANDATORY for all modules
- Standard: RFC 6762 (mDNS) + RFC 6763 (DNS-SD)
- Every ITxPT module with an IP address MUST announce itself via SRV + TXT records
- Domain: always `.local`

**SRV Record format:**
```
<UniqueId>_<name>._<type>._<protocol>.local  TTL class SRV priority weight port target
```

Field rules:
- `UniqueId`: arbitrary unique string, must NOT contain `_`
- `name`: service name (inventory, time, gnsslocation, fmstoip, vehicletoip, avms, mqtt-broker…)
- `type`: `itxpt_http` | `itxpt_multicast` | `itxpt_socket` | `sntp` | `mqtt`
- `protocol`: `_tcp` or `_udp` only
- All fields: lowercase letters, digits, hyphens only; max 14 characters per field

**Mandatory TXT record fields (all services):**
```
txtvers=1
version=<spec version, e.g. 2.3.0>
release=Linden
swvers=<your software version>
manufacturer=<your company>
atdatetime=<ISO 8601 UTC, YYYY-MM-DDTHH:MM:SSZ>
interval=<update frequency minutes, 1–30>
```

### Transport Protocols

| SRV Type           | Transport        | Port   | Use                          |
|--------------------|------------------|--------|------------------------------|
| _itxpt_socket      | TCP socket       | 9      | Low-level ITxPT socket       |
| _itxpt_multicast   | UDP multicast    | varies | GNSS, VEHICLEtoIP, FMStoIP   |
| _itxpt_http        | HTTP/1.1 TCP     | varies | AVMS, Inventory, FMStoIP cfg |
| sntp               | UDP unicast      | 123    | Time service                 |
| mqtt               | TCP              | 1883   | MQTT broker                  |

**Multicast address**: `239.255.42.21` (all ITxPT UDP multicast services)

**IP addressing**: private IPv4 range `192.168.x.0/24` (Class C); DHCP or static

### HTTP Service Rules
- HTTP 1.1 only (not 1.0, not 2.0)
- `Content-Type: application/xml` or `application/json`
- `Content-Length` header required in all requests and responses
- Subscription model: client POSTs subscribe request with callback URL (ClientIPAddress, ReplyPort, ReplyPath); service POSTs data deliveries to client
- Unsubscribe: client POSTs unsubscribe request to same endpoint
- `Connection: close` or `Connection: keep-alive` for delivery connections
- HTTP 501 if optional operation not implemented

### DateTime Format
All datetime values in TXT records and XML payloads:
- Format: `YYYY-MM-DDTHH:MM:SSZ` (e.g. `2024-08-13T14:22:11Z`)
- Always UTC — no local timezone in delivered data

---

## S02P01 — Module Inventory Service

- **Mandatory**: YES — every IP-addressed module
- **Service name**: `inventory`
- **Port**: 80 (HTTP) or 9 (socket)
- **SRV type**: `_itxpt_http._tcp` or `_itxpt_socket._tcp`

**15 Mandatory TXT fields:**

| Field            | Description                                      | Example                        |
|------------------|--------------------------------------------------|--------------------------------|
| txtvers          | Always 1                                         | txtvers=1                      |
| version          | ITxPT spec version                               | version=2.2.0                  |
| release          | Release name                                     | release=Linden                 |
| swvers           | Your software version                            | swvers=1.3.4                   |
| type             | Module type (VCG, GNSS, AVMS, etc.)             | type=VCG                       |
| model            | Manufacturer model name                          | model=model26                  |
| manufacturer     | Manufacturer name                                | manufacturer=companyABC        |
| serialnumber     | Module serial number                             | serialnumber=SN0123456789      |
| softwareversion  | Installed software version                       | softwareversion=v1.2.3         |
| hardwareversion  | Hardware version                                 | hardwareversion=revision D     |
| macaddress       | Primary MAC (colon-separated HEX)                | macaddress=CF:DA:98:63:9D:F6   |
| status           | Self-test result (0=OK, non-zero=error)          | status=0                       |
| services         | Semicolon-separated list of running services     | services=inventory;time;gnss   |
| atdatetime       | Timestamp of this record                         | atdatetime=2024-08-13T14:22:11Z|
| interval         | Update frequency minutes (1–30)                  | interval=10                    |

**Optional TXT fields**: `xstatus` (16-char hex, 2 bits per item: 00=OK, 01=Alarm, 10=Warning, 11=N/A), `submodules` (boolean), `path` (URL path to XML)

---

## S02P02 — Time Service

- **Mandatory**: YES (buses)
- **Service name**: `time`
- **Port**: 123 (SNTP UDP)
- **SRV type**: `_sntp._udp`
- Standard: RFC 5905 (SNTP)
- UTC only — no timezone conversion
- Only ONE Time service instance per installation
- If no time source available, service MUST NOT be announced
- Time source: NTP server or GNSS Date-Time elements

**SRV Record:**
```
[UniqueId]_time._sntp._udp.local 120 IN SRV 0 0 123 [hostname]
```

---

## S02P03 — GNSS Location Service

- **Mandatory**: YES (buses)
- **Service name**: `gnsslocation`
- **Port**: 14005 (UDP multicast)
- **SRV type**: `_itxpt_multicast._udp`
- Only ONE GNSSLocation service per installation
- Recommended update rate: 1/s or 2/s
- Must use multi-constellation receiver (GPS + Galileo minimum)

**SRV Record:**
```
[UniqueId]_gnsslocation._itxpt_multicast._udp.local 120 IN SRV 0 0 14005 [hostname]
```

**Additional mandatory TXT fields**: `multicast=239.255.42.21`, `frequency=1/1`

**Mandatory XML payload fields (v2.3.0):**
- `GNSSLocation.Latitude.Degree` — double [-90.0, +90.0]; NULL if SignalQuality=NotValid
- `GNSSLocation.Latitude.Direction` — N or S
- `GNSSLocation.Longitude.Degree` — double [-180.0, +180.0]
- `GNSSLocation.Longitude.Direction` — W or E
- `GNSSLocation.SignalQuality` — `aGPS | dGPS | Estimated | GPS | NotValid | Unknown | PPS | RTK | Float-RTK | Manual | Simulation`
- `GNSSLocation.Fix` — `NoFix | 2D | 3D | DR | 3D+DR`
- `GNSSLocation.GNSSTypes.GNSSTypeName` — `GPS | Glonass | Galileo | Beidou | IRNSS | Other | DeadReckoning`

**BREAKING CHANGE v2.3.0**: `GNSSType` (singular) is deprecated; use `GNSSTypes.GNSSTypeName` (plural)

**Optional fields**: Altitude (only when Fix=3D/3D+DR), Time, Date, SpeedOverGround, NumberOfSatellites, HDOP, VDOP, TrackDegreeTrue, GNSSCoordinateSystem, Raw NMEA frames

---

## S02P04 — FMStoIP Service

- **Mandatory**: YES (thermal/electric buses with CAN-FMS)
- **Service name**: `fmstoip`
- Publishes TWO services: HTTP config + UDP multicast provision
- Reads Bus-FMS CAN (J1939) — read-only (only OEM can write to FMS CAN)

**HTTP Config Service:**
- Port: 8085
- SRV: `[UniqueId]_fmstoip._itxpt_http._tcp.local 120 IN SRV 0 0 8085 [hostname]`
- Operations: addpgn, removepgn, getpgn, addspndecode, registrationstatus, pgnstatus, pgninfoknown, pgninfo

**Mandatory TXT fields (additional):**
- `path=fmstoip/`
- `operations=addpgn;removepgn;getpgn;...`
- `cachestrat=fmsstd` (values: registered | fmsstd | all)
- `fmsstdbus=4` (Bus-FMS standard version)
- `fmsstdtruck=4`
- `fmsstdsupport=4;5` (semicolon-separated supported versions)
- `spndecode=none` (values: none | fms | J1939DA_YYYYMM)

**Provision Service**: joins multicast group `239.255.42.21`

---

## S02P05 — VEHICLEtoIP Service

- **Mandatory**: YES (rail/tram); optional (buses)
- **Service name**: `vehicletoip`
- **Port**: 15030 (UDP multicast)
- **SRV type**: `_itxpt_multicast._udp`
- Update rate: max 1/s

**SRV Record:**
```
[UniqueId]_vehicletoip._itxpt_multicast._udp.local 120 IN SRV 0 0 15030 [hostname]
```

**Mandatory XML payload fields**: RecordedAtTime, Odospeed, Ododistance, Odofault, DoorsStatus, ActiveCabin, PresenceHV, PropulsionSystemActive, StopRequest, VehicleType, VehicleNumber, VehicleMode, VehicleOrder, ITSPower

**DoorsStatus values**: `AllDoorsClosedLocked | DoorUnlocked | DoorOpen`
**VehicleType values**: `Bus | Tram | Ferry | Light Rail | Heavy Rail`

---

## S02P06 — AVMS Service

- **Mandatory**: Optional (but if implemented, 4 operations are mandatory)
- **Service name**: `avms`
- **Port**: 8000 (HTTP/TCP)
- **SRV type**: `_itxpt_http._tcp`

**SRV Record:**
```
[UniqueId]_avms._itxpt_http._tcp.local 120 IN SRV 0 0 8000 [hostname]
```

**Mandatory TXT fields (additional):** `path=avms/`, `operations=runmonitoring;plannedpattern;...`

**Operations:**

| Operation           | Path                        | Mandatory? |
|---------------------|-----------------------------|------------|
| Run Monitoring      | avms/runmonitoring          | MANDATORY  |
| Planned Pattern     | avms/plannedpattern         | MANDATORY  |
| Vehicle Monitoring  | avms/vehiclemonitoring      | MANDATORY  |
| Journey Monitoring  | avms/journeymonitoring      | MANDATORY  |
| Pattern Monitoring  | avms/patternmonitoring      | Optional   |
| General Message     | avms/generalmessage         | Optional   |
| Connection Monitoring | avms/connectionmonitoring | Optional   |

**CRITICAL — Journey State Model**: Must use **departure-triggered** model (not SIRI arrival-only):
- MonitoredCallRef → PreviousCallRef on vehicle departure
- First OnwardCallRef → MonitoredCallRef on departure
- OnwardCalls set shrinks by one per stop departure

**RunState values**: `DeadRun | RunToPattern | RunPattern`
**Language codes**: ISO 639-3 (3-letter codes) on `language` XML attribute

---

## S02P09 — MQTT Broker Service

- **Mandatory**: Optional
- **Service name**: `mqtt-broker`
- **Port**: 1883 (TCP)
- **SRV type**: `_mqtt._tcp`
- MQTT version: **5.0 mandatory**; 3.1.1 optional
- Security: SSL/TLS required, ACLs, username/password, client + server certificates

**SRV Record:**
```
[UniqueId]_mqtt-broker._mqtt._tcp.local 120 IN SRV 0 0 1883 [hostname]
```

**Additional mandatory TXT fields:**
- `brand=<broker brand, e.g. mosquitto>`
- `proto=5.0` or `proto=3.1.1;5.0`
- `topic=itxpt/` (root topic prefix)
- `listeners=1883;8883` (all active MQTT port listeners)
- `websocket=9005;9010` (or `9` if not supported)

**MQTT Topic Rules:**
- Max 256 characters
- Allowed: ASCII lowercase letters, numbers, underscore `[0-9, a-z, _]`
- Must NOT begin with underscore or digit
- Separate internal topics from back-office topics using edge bridge

---

## S03 — Back-Office Requirements

### Network Architecture
- **PTO IP Network**: private IP network for all operator systems; supports WAN (3G/4G/5G, VPN)
- Components: AVMS back-office, Passenger Information back-office, Telediagnostic back-office

### Security
- TLS 1.2 minimum for all public network connections
- VPN/IPSEC or private APN for vehicle-to-back-office links
- Segregate Onboard Passenger IP Network from Backbone (DMZ, VLAN, proxy, NAT)

### Standards Required
- **SIRI**: Service Interface for Real-time Information (CEN standard)
- **NeTEx**: Network and Timetable Exchange (CEN standard)
- **Transmodel**: CEN conceptual data model (base for SIRI/NeTEx)
- **TiGR**: Telediagnostic Gateway and Repository (S03P01)

---

## S01 — Hardware Quick Reference

**Power connector**: MCP 6-pin, TYCO 1-965641-1 Blue Code A (vehicle side)
- PIN 1: 24V Continuous (B+)
- PIN 2: Power Ground
- PIN 3: 24V Active (+15/Ignition)
- PIN 4: Full Power Available (source type, active HIGH)
- PIN 5: Reserved
- PIN 6: 24V Passive (+30)

**ECO Power Modes**: ECO0 (full operation), ECO1 (full features, low-power peripherals), ECO2 (data backup only), SLEEP (no ITS network features)

**Mandatory signals**: Low battery (sink, active LOW, 120s warning), Full power available (source, active HIGH), Awake request, Door 1–6 open (source, active HIGH), Stop Request (source, active HIGH)

---

## Service Port Reference

| Service         | DNS Name       | Type             | Port  | Transport    | Mandatory (Bus)     |
|-----------------|----------------|------------------|-------|--------------|---------------------|
| Inventory       | inventory      | itxpt_http       | 80    | TCP          | YES                 |
| Inventory       | inventory      | itxpt_socket     | 9     | TCP          | YES                 |
| Time            | time           | sntp             | 123   | UDP          | YES                 |
| GNSS Location   | gnsslocation   | itxpt_multicast  | 14005 | UDP mcast    | YES                 |
| FMStoIP config  | fmstoip        | itxpt_http       | 8085  | TCP          | YES (thermal/elec.) |
| FMStoIP data    | fmstoip        | itxpt_multicast  | cfg'd | UDP mcast    | YES (thermal/elec.) |
| VEHICLEtoIP     | vehicletoip    | itxpt_multicast  | 15030 | UDP mcast    | YES (rail/tram)     |
| AVMS            | avms           | itxpt_http       | 8000  | TCP          | Optional            |
| APC             | apc            | itxpt_http       | cfg'd | TCP          | Optional            |
| MADT            | madt           | itxpt_http       | cfg'd | TCP          | Optional            |
| MQTT Broker     | mqtt-broker    | mqtt             | 1883  | TCP          | Optional            |

---

## Key Gotchas / Common Mistakes

1. **`GNSSType` vs `GNSSTypes`**: v2.3.0 deprecated the singular `GNSSType` field; use `GNSSTypes.GNSSTypeName`
2. **Release field**: MUST be `release=Linden` (not a version number) in every TXT record
3. **HTTP version**: HTTP 1.1 only — not HTTP/2
4. **AVMS state model**: departure-triggered, not arrival-triggered
5. **MQTT version**: 5.0 is mandatory (not just 3.1.1)
6. **Time service**: if no time source, do NOT announce the service
7. **Only one SNTP server** and **one GNSSLocation service** per installation
8. **FMStoIP publishes two separate DNS-SD records** (HTTP config + UDP multicast)
9. **UniqueId in SRV records** must not contain underscores
10. **Content-Length header** is required in all HTTP requests and responses
