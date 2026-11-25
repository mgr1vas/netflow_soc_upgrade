# Nikto Cheatsheet & Lab Reference

> **Lab Safety Notice**
>
> Only run `nikto` against web services you own or have explicit authorization to test (such as `127.0.0.1` or the isolated university lab VM). Avoid Denial of Service checks (`-Tuning x6`) on shared or production servers.

## Overview & NetFlow SOC Context

**Nikto** is an open-source **Layer 7 (Application Layer)** web server scanner. It checks HTTP/HTTPS endpoints for outdated server software, dangerous default files, misconfigured security headers, exposed directories, and common web vulnerabilities.

In a **NetFlow-based SOC pipeline** (`GoFlow2` → `ClickHouse` → `Grafana`):

* **NetFlow does not inspect HTTP payloads, URLs, or headers.** All detailed vulnerability findings appear in Nikto's own report output (`txt`, `csv`, `json`, `htm`).
* **At the flow level**, `nikto` produces a sustained, high-frequency burst of TCP flows targeting a **single destination port** (e.g., `80`, `443`, `3000`, or `8080`). This makes it useful for testing **single-port connection spike** alerts while simultaneously auditing the web endpoints (like `flow_bridge.py` or Grafana).

## Installation (Debian / Ubuntu)

```bash
sudo apt update && sudo apt install -y nikto
```

## Core Targeting & Authentication

| Flag | Description | Example |
| :--- | :--- | :--- |
| `-h <target>` | Target IP, hostname, or URL | `nikto -h http://127.0.0.1` |
| `-p <ports>` | Target port(s) (defaults to `80`) | `-p 3000,8080` |
| `-ssl` | Force HTTPS/SSL mode on the port | `nikto -h 127.0.0.1 -p 443 -ssl` |
| `-nossl` | Disable SSL checks (plain HTTP only) | `nikto -h 127.0.0.1 -p 8080 -nossl` |
| `-id <user:pass>` | HTTP Basic Auth credentials | `-id admin:labpass` |
| `-vhost <host>` | Specify a custom `Host` header | `-vhost lab.local` |

## Scan Tuning (`-Tuning`)

Use `-Tuning` to run only specific check categories. Prefix with `x` to **exclude** categories instead (for example, `-Tuning x6` runs everything except Denial of Service tests).

| Code | Test Category |
| :--- | :--- |
| `0` | File upload checks |
| `1` | Interesting files / seen in logs |
| `2` | Misconfiguration / default files |
| `3` | Information disclosure |
| `4` | Injection (XSS / Script / HTML) |
| `5` | Remote file retrieval (inside web root) |
| `6` | Denial of Service *(exclude on live servers with `x6`)* |
| `7` | Remote file retrieval (server-wide) |
| `8` | Command execution / remote shell |
| `9` | SQL injection |
| `a` | Authentication bypass |
| `b` | Software identification |
| `c` | Remote source inclusion |

## Rate Control, Behavior & Output

| Flag | Description |
| :--- | :--- |
| `-maxtime <time>` | Hard stop after a set duration (e.g., `30s`, `60s`, `2m`) |
| `-Pause <sec>` | Delay in seconds between requests (throttles traffic noise) |
| `-timeout <sec>` | Per-request timeout in seconds (default `10`) |
| `-no404` | Skip 404 heuristic guessing (faster on custom/minimal APIs) |
| `-useragent "<str>"` | Set a custom `User-Agent` header |
| `-evasion <1-8>` | Apply IDS evasion encoding (e.g., `1` = random URI encoding) |
| `-Display <opts>` | Control console output (`V` verbose, `1` redirects, `2` cookies, `3` 200/OK responses) |
| `-o <file>` | Output filename |
| `-Format <fmt>` | Output format (`txt`, `csv`, `htm`, `xml`, `json`) |

## Ready-to-Run Lab Commands

```bash
# 1. Standard scan against the legacy Python HTTP bridge on port 8080
nikto -h http://127.0.0.1 -p 8080 -nossl

# 2. Time-boxed 60-second burst against Grafana (port 3000) to test SOC flow spikes
nikto -h http://127.0.0.1 -p 3000 -nossl -maxtime 60s

# 3. Safe misconfiguration & info-disclosure audit (skips DoS tests) exported to CSV
nikto -h http://127.0.0.1 -p 8080 -Tuning 123b -o nikto_audit.csv -Format csv

# 4. Run all checks except Denial of Service and export to HTML report
nikto -h http://127.0.0.1 -p 3000 -Tuning x6 -o nikto_report.htm -Format htm

# 5. Throttled HTTPS scan with a 1-second pause between requests
nikto -h https://127.0.0.1 -p 443 -ssl -Pause 1

# 6. Scan an authenticated endpoint (e.g., ClickHouse HTTP interface)
nikto -h http://127.0.0.1 -p 8123 -id admin:labpass -no404 -maxtime 30s
```

## What to Look for in ClickHouse / Grafana

When running `nikto` while monitoring flows in the upgraded SOC pipeline:

* **Single-Port Flow Surge:** Sharp spike in flow count where `DstPort` is fixed (e.g., `3000` or `8080`) and `Proto = 6` (TCP).
* **Asymmetric Traffic Volume:** Steady outbound requests from the scanner paired with fluctuating response `Bytes` from the target web server.
* **Complementary Layer-7 Findings:** Pair the Grafana spike timeline with Nikto's `nikto_audit.csv` output to document both the **network anomaly** (NetFlow) and the **application-layer posture** (missing headers, server banners, exposed endpoints).