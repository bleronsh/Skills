# ITxPT Skills for Claude Code

Two [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills that bring ITxPT (Information Technology for Public Transport) specification knowledge and compliance auditing directly into your development workflow.

Built for the **Linden** release of the ITxPT specification, covering S01 (Hardware), S02 (Onboard Services), and S03 (Back-Office) requirements.

## Skills

### `itxpt-knowledge` — Specification Knowledge Base

**Type:** Context (auto-loaded)

A persistent knowledge base covering the full ITxPT Linden release. Once installed, it is automatically available in every conversation — no need to re-read the 13 source PDFs each session.

**What it covers:**

| Spec | Area |
|------|------|
| S01 | Installation & hardware requirements |
| S02P00 | Networks, protocols, DNS-SD, mDNS |
| S02P01 | Module Inventory service |
| S02P02 | Time service (SNTP) |
| S02P03 | GNSS Location service |
| S02P04 | FMStoIP service |
| S02P05 | VEHICLEtoIP service |
| S02P06 | AVMS service |
| S02P09 | MQTT Broker service |
| S03 | Back-office requirements |

Includes service port reference tables, mandatory TXT record fields, XML payload schemas, and common gotchas.

---

### `itxpt-check` — Compliance Checker

**Type:** Prompt (user-invocable)

An interactive compliance auditor that checks a service, component, file, or code snippet against ITxPT Linden release requirements. It runs through 79 structured checks and produces a detailed report.

**What it checks:**

- DNS-SD service announcement (SRV records, mDNS)
- TXT record mandatory fields (txtvers, version, release, swvers, etc.)
- Service-specific requirements (Inventory, Time, GNSS, FMStoIP, VEHICLEtoIP, AVMS, MQTT)
- HTTP protocol compliance (version, headers, subscription model)
- DateTime format validation (ISO 8601 UTC)
- Back-office requirements (TLS, network segregation, SIRI/NeTEx)

**Output format:**

Each check is marked as PASS, FAIL, WARN, or N/A, followed by a summary table and a priority fix list sorted critical-first.

## Installation

Install both skills into your project using the Claude Code CLI:

```bash
claude skill add --from "https://github.com/bleronsh/Skills" itxpt-knowledge
claude skill add --from "https://github.com/bleronsh/Skills" itxpt-check
```

Or install them globally (available in all projects):

```bash
claude skill add -g --from "https://github.com/bleronsh/Skills" itxpt-knowledge
claude skill add -g --from "https://github.com/bleronsh/Skills" itxpt-check
```

## Usage

### Knowledge base

The `itxpt-knowledge` skill loads automatically as context. Just ask questions naturally:

```
> What port does the GNSS Location service use?
> What TXT fields are mandatory for the Inventory service?
> How does the AVMS journey state model work?
```

### Compliance checker

Invoke the checker with `/itxpt-check` followed by what you want to audit:

```
> /itxpt-check src/services/gnss-location.ts
> /itxpt-check our AVMS service implementation
> /itxpt-check the MQTT broker configuration
```

The checker will read the target, run all applicable checks, and produce a structured audit report with a priority fix list.

## Example Output

```
### Compliance Audit Results: GNSS Location Service

| # | Check                              | Result | Notes                        |
|---|------------------------------------|--------|------------------------------|
| 1 | SRV record announced via mDNS     | PASS   |                              |
| 2 | SRV format correct                 | PASS   |                              |
| 25| multicast=239.255.42.21 in TXT     | FAIL   | Field missing from TXT record|
| 29| Uses GNSSTypes.GNSSTypeName        | WARN   | Still using deprecated GNSSType |
...

### Summary
- PASS: 24 checks
- FAIL: 3 checks
- WARN: 1 check
- N/A: 51 checks

### Priority Fix List
1. [CRITICAL] — multicast TXT field: must include multicast=239.255.42.21
2. [CRITICAL] — GNSSTypes: migrate from deprecated GNSSType to GNSSTypes.GNSSTypeName (v2.3.0)
...
```

## Spec Versions

These skills are based on the following ITxPT specification versions:

| Spec | Version |
|------|---------|
| S01 | v2.3.2 |
| S02P00 | v2.2.1 |
| S02P01 | v2.2.0 |
| S02P02 | v2.2.1 |
| S02P03 | v2.3.0 |
| S02P04 | v2.3.0 |
| S02P05 | v2.2.1 |
| S02P06 | v2.3.0 |
| S02P07 | v2.2.2 |
| S02P08 | v3.0.1 |
| S02P09 | v2.3.1 |
| S03P00 | v3.0.0 |
| S03P01 | v3.0.2 |

## Contributing

Contributions are welcome. If you find a spec inaccuracy or want to add checks for additional ITxPT services (APC, MADT), open an issue or submit a pull request.

## License

MIT License — see [LICENSE](LICENSE) for details.
