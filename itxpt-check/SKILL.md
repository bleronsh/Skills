---
name: itxpt-check
type: prompt
description: >
  ITxPT compliance checker for the YES-EU Fleet Management Platform.
  Audits a service, component, file, or code snippet against ITxPT
  Linden release requirements and reports compliance gaps.
user-invocable: true
arguments: true
---

# ITxPT Compliance Checker

You are performing a structured ITxPT compliance audit for the YES-EU Fleet Management Platform.

The target to audit is: **$ARGUMENTS**

If `$ARGUMENTS` specifies a file path, read that file first. If it names a service or component, use the knowledge from the ITxPT knowledge skill (itxpt-knowledge) and the spec facts below.

---

## ITxPT Compliance Audit — Required Checks

Work through each applicable section below. For each check, output one of:
- ✅ **PASS** — requirement is met
- ❌ **FAIL** — requirement is missing or wrong (explain what is required)
- ⚠️ **WARN** — partially met or unclear (explain what to verify)
- ➖ **N/A** — not applicable to this component

At the end, output a **Summary Table** and a **Priority Fix List** (FAIL items only, sorted critical-first).

---

## Section 1 — DNS-SD Service Announcement

For any service that announces itself on the network:

1. Is a SRV record published via mDNS (RFC 6762/6763)?
2. Does the SRV record follow the format `<UniqueId>_<name>._<type>._<protocol>.local`?
3. Is the `UniqueId` free of underscores (`_`)?
4. Are all SRV record fields lowercase, max 14 chars, using only `[a-z0-9-]`?
5. Is the `type` one of: `itxpt_http`, `itxpt_multicast`, `itxpt_socket`, `sntp`, `mqtt`?
6. Is the `protocol` either `_tcp` or `_udp` (not both, not other)?

---

## Section 2 — TXT Record — Universal Mandatory Fields

Every ITxPT service TXT record must include ALL of these:

| Field | Required Value |
|---|---|
| txtvers | integer, always `1` |
| version | ITxPT spec version string, e.g. `2.3.0` |
| release | Must be exactly `Linden` |
| swvers | Your software version string |
| manufacturer | Your company name |
| atdatetime | ISO 8601 UTC: `YYYY-MM-DDTHH:MM:SSZ` |
| interval | Integer, 1–30 (minutes between updates) |

Check each field:
7. `txtvers=1` present?
8. `version` present and matches expected spec version?
9. `release=Linden` present (not a version number)?
10. `swvers` present?
11. `manufacturer` present?
12. `atdatetime` present and in correct ISO 8601 UTC format?
13. `interval` present and in range 1–30?

---

## Section 3 — Service-Specific Requirements

Identify which ITxPT service(s) this component implements, then check the relevant sub-sections below.

### 3A — Inventory Service (S02P01)
*Mandatory for every IP-addressed module.*

14. SRV type is `_itxpt_http._tcp` (port 80) or `_itxpt_socket._tcp` (port 9)?
15. All 15 mandatory TXT fields present? (`txtvers`, `version`, `release`, `swvers`, `type`, `model`, `manufacturer`, `serialnumber`, `softwareversion`, `hardwareversion`, `macaddress`, `status`, `services`, `atdatetime`, `interval`)
16. `macaddress` is colon-separated HEX (`XX:XX:XX:XX:XX:XX`)?
17. `status` is integer (0=OK)?
18. `services` is semicolon-separated list of active service names?
19. `type` is a valid module type enum (VCG, GNSS, AVMS, etc.)?

### 3B — Time Service (S02P02)
*Mandatory for buses.*

20. SRV type is `_sntp._udp`, port 123?
21. Time delivered in UTC only (no timezone conversion)?
22. If no time source available, is the service NOT announced?
23. Is there only ONE Time service instance per installation?

### 3C — GNSS Location Service (S02P03)
*Mandatory for buses.*

24. SRV type is `_itxpt_multicast._udp`, port 14005?
25. TXT record includes `multicast=239.255.42.21`?
26. TXT record includes `frequency` field?
27. XML payload includes mandatory `Fix` field?
28. XML payload includes mandatory `SignalQuality` field?
29. XML payload uses `GNSSTypes.GNSSTypeName` (NOT deprecated `GNSSType`)?
30. `Fix` value is one of: `NoFix | 2D | 3D | DR | 3D+DR`?
31. `SignalQuality` value is one of: `aGPS | dGPS | Estimated | GPS | NotValid | Unknown | PPS | RTK | Float-RTK | Manual | Simulation`?
32. When `SignalQuality=NotValid`, are Latitude/Longitude set to NULL?
33. `Altitude` is only included when `Fix=3D` or `Fix=3D+DR`?
34. Is there only ONE GNSSLocation service per installation?

### 3D — FMStoIP Service (S02P04)
*Mandatory for thermal/electric buses with CAN-FMS.*

35. Are TWO separate DNS-SD records published (HTTP config + UDP multicast provision)?
36. HTTP config SRV is `_itxpt_http._tcp`, port 8085?
37. TXT record includes `path=fmstoip/`?
38. TXT record includes `cachestrat` (one of: `registered`, `fmsstd`, `all`)?
39. TXT record includes `fmsstdbus` and `fmsstdtruck` version fields?
40. TXT record includes `fmsstdsupport` (semicolon-separated versions)?
41. TXT record includes `spndecode` (one of: `none`, `fms`, `J1939DA_YYYYMM`)?
42. Provision service joins multicast group `239.255.42.21`?

### 3E — VEHICLEtoIP Service (S02P05)
*Mandatory for rail/tram; optional for buses.*

43. SRV type is `_itxpt_multicast._udp`, port 15030?
44. TXT record includes `multicast=239.255.42.21`?
45. XML payload update rate does not exceed 1/s?
46. All mandatory fields present: `RecordedAtTime`, `Odospeed`, `Ododistance`, `Odofault`, `DoorsStatus`, `ActiveCabin`, `PresenceHV`, `PropulsionSystemActive`, `StopRequest`, `VehicleType`, `VehicleNumber`, `VehicleMode`, `VehicleOrder`, `ITSPower`?
47. `DoorsStatus` is one of: `AllDoorsClosedLocked | DoorUnlocked | DoorOpen`?
48. `VehicleType` is one of: `Bus | Tram | Ferry | Light Rail | Heavy Rail`?

### 3F — AVMS Service (S02P06)
*Optional service; if implemented, mandatory operations apply.*

49. SRV type is `_itxpt_http._tcp`, port 8000?
50. TXT record includes `path=avms/`?
51. TXT record includes `operations` field listing all implemented operations?
52. All four mandatory operations implemented: `runmonitoring`, `plannedpattern`, `vehiclemonitoring`, `journeymonitoring`?
53. Non-implemented optional operations return HTTP 501?
54. Journey state model is **departure-triggered** (not arrival-triggered)? i.e. MonitoredCallRef → PreviousCallRef on departure, first OnwardCallRef → MonitoredCallRef on departure?
55. Language codes use ISO 639-3 (3-letter codes)?

### 3G — MQTT Broker Service (S02P09)
*Optional service.*

56. SRV type is `_mqtt._tcp`, port 1883?
57. MQTT version 5.0 supported (mandatory)?
58. SSL/TLS enabled?
59. ACL (Access Control Lists) configured?
60. Username/password authentication available?
61. Client + server certificates supported?
62. TXT record includes `brand`, `proto`, `topic`, `listeners`, `websocket`?
63. `proto` includes `5.0`?
64. `topic` ends with `/` (e.g. `topic=itxpt/`)?
65. MQTT topics are lowercase, start with a letter (not `_` or digit), max 256 chars?

---

## Section 4 — HTTP Protocol Requirements
*For any service using `_itxpt_http`.*

66. HTTP version is 1.1 (not 1.0, not HTTP/2)?
67. `Content-Type` is `application/xml` or `application/json`?
68. `Content-Length` header included in all requests and responses?
69. Subscription endpoint accepts POST with `ClientIPAddress`, `ReplyPort`, `ReplyPath`?
70. Subscribe response returns HTTP 200 with `<SubscribeResponse><Active>true</Active></SubscribeResponse>`?
71. Data deliveries are POSTed to client callback URL?
72. Unsubscribe is handled via POST to same endpoint?

---

## Section 5 — DateTime Format

73. All datetime values use ISO 8601 UTC format `YYYY-MM-DDTHH:MM:SSZ`?
74. No local timezone offsets in service data payloads?

---

## Section 6 — Back-Office (S03) — if applicable

75. Vehicle-to-back-office communication uses TLS 1.2+ or VPN/IPSEC?
76. Passenger network is segregated from Backbone (DMZ/VLAN/NAT)?
77. SIRI-compatible real-time data ingestion from AVMS?
78. NeTEx format supported for timetable/network data exchange?
79. TiGR (S03P01) telediagnostic interface implemented?

---

## Output Format

After completing all applicable checks, produce:

### Compliance Audit Results: [component name]

**Audit scope**: [list which sections were checked]

| # | Check | Result | Notes |
|---|---|---|---|
| 1 | SRV record announced via mDNS | ✅/❌/⚠️/➖ | |
| … | … | … | |

### Summary

- ✅ Passing: X checks
- ❌ Failing: X checks
- ⚠️ Warnings: X checks
- ➖ Not applicable: X checks

### Priority Fix List

List all FAIL items in priority order (critical spec violations first, then optional):

1. **[CRITICAL]** — [check name]: [what is required and what was found]
2. …

### Recommendation

One paragraph summarizing overall compliance posture and the most important next steps.
